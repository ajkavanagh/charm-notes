# Syncing Fernet Tokens

Due to a rather nasty
bug [(LP: #1849519)](https://bugs.launchpad.net/charm-keystone/+bug/1849519),
the way that Fernet tokens are synced between machines has changed.

## NOTE: a lot of this is now 'obsolete' thinking - might want to do it with vault

i.e. use vault to hold the keys and thus be accessible around the system.  This
would enable a 'simpler' distribution system that doesn't rely on juju or
ssh/scp between the units.  Determining the leader would need to be a Juju
thing, although it might be possible to do this via some kind of voting thing
in vault.  Tricky!

## Original Implementation

The original implementation did the following:

  * A regular intervals (determined by config, and written by a Context/tempate)
    a cron job called juju run on a script.
  * This script then called into the the function `fernet_keys_rotate_and_sync`
    in `hooks/keystone_utils.py` which then checked if the unit was the leader
    and if fernet keys are enabled.  If not then it exits.
  * If it is leader and fernet keys are enabled, then it checks to see if
    (according to the config) sufficient time has elapsed for the tokens to be
    rotated.
  * If so, the the tokens are rotated using `fernet_rotate`.
  * And then they are synced using the leader_settings feature of Juju.

On the non-leaders, when the leader-settings hook fires, it writes the keys in
the leader-settings to the appropriate place.

This, in theory, should keep the keys in sync.

However, it doesn't.  It turns out that keys can go out of sync and *stay* out
of sync for at least the period of rotation (default of 1 day/86000 seconds).
This needed to be resolved.

## Issues with the original implementation

The main issues with the original implementation were:

  * The synchronisation between the leader and remaining units can fail.  It's
    not clear why, but there are issues from customers where key rotation fails
    for at least 24h.
  * There is no facility to cause a resync if a keystone unit goes out of sync
    (for any reason).  This is because the code only attempts to sync keys
    during a rotation which is (by default) 86000 seconds.

## Revised Solution

The key features that the revised solution must have are:

  * Key rotations must only happen on the leader and the leaders keys (if they
    change) must be represented on keystone units that are part of the HA
    cluster.
  * The time between a key rotation on the leader and those keys appearing on
    the non-leaders must be kept short.
  * Network failures during key-rotations must restore keys to those units as
    quickly as possible.
  * Leadership changes should not impact key synchronisations.  i.e. all units
    should still be synchronised to the new leader.
  * When copying/replacing key files they should be atomic operations; this
    implies they should be copied to a temporary file and then renamed.
  * The system should keep working even if juju is completely disabled.  i.e.
    rotations and synchronisations should continue.

The revised solution uses a direct network sync approach to write files to the
target keystone non-leader units.  The approach is:

On the leader:

  * A cron job will be installed/run that calls the
    `fernet_keys_rotate_and_sync` function as existing.  This runs every `n`
    minutes.
  * The check for rotating keys is performed (e.g. is leader and is configured
    for it) and if so, the keys are rotated.
  * The sync function is called _regardless_ of whether the keys were rotated.

The sync function attempts to write the keys to non-leader units.  It needs to
keep a record of which keystone units were successfully synced, and thus, which
ones need to be resynced on the next attempt.


## Fernet Token implementation key notes

Key implementation notes:

  * When leadership changes, the leader needs to determine if it should rotate
    the keys in addition to forcing a sync to the other units.  This implies
    that a new leader needs to determine roughly how long it was since the last
    sync by the leader, which means that the time of that sync needs to be
    recorded.  This is based on the keystone units all being in time sync (to
    within the resolution of half the sync time), which should be a reasonable
    assumption.  Thus, the key rotation time needs to be synced as well.
  * The leader needs to keep a record of which keystone units have been
    successfully synced and retry at regular intervals if a unit is out of
    sync.  This implies that the leader needs to keep track of which units have
    been syned.
  * When a new leader is elected, it needs to 'forget' about which units have
    been synced, and force sync them with its keys.

The master has the keys in:

CREDENTIAL_KEY_REPOSITORY = '/etc/keystone/credential-keys/'
FERNET_KEY_REPOSITORY = '/etc/keystone/fernet-keys/'

And they need to be copied, along with a timestamp, to the non-leader units.
The timestamp for the rotation is stored in Juju unitdata storage (KV) with the
key `last-fernet-key-rotation-timestamp` in ISO8601 in UTC 0 format (no
offset).  The units that have been successfully synced are stored in
a JSON list at Juju unitdata storate with key `fernet-key-synced-to` in the
form `[<keystone-unit>]`.


## Code segments

This wasn't used by gives an idea of what was going to be in the fix.

```python
def sync_fernet_tokens():
    """Read current key sets and sync them (as needed) to non-leader units.

    This function should only be called on a leader keystone unit.  Otherwise
    bad things will happen.

    The keys are read from the `FERNET_KEY_REPOSITORY` and
    `CREDENTIAL_KEY_REPOSITORY` directories.

    This function logs if it can't sync the tokens but does not fail.  Other
    units could be offline, stopped, rebooting, of just network partitioned.
    They will be recovered and synced on a future cron job.
    """
    if not is_leader():
        return
    db = unitdata.kv()
    units_already_synced = db.get(FERNET_KEYS_SYNCED_UNITS) or []
    unit_states = get_peers_unit_state()
    to_sync_units = set(unit_states.keys()).difference(units_already_synced)
    if to_sync_units:
        cmds = (("tar czf /tmp/fernet.tgz -C {source} {source}; "
                 "sudo chown ubuntu:ubuntu /tmp/fernet.tgz"
                 .format(source=FERNET_KEY_REPOSITORY)),
                ("tar czf /tmp/credentials.tgz -C {source} {source}; "
                 "sudo chown ubuntu:ubuntu /tmp/credentials.tgz"
                 .format(source=CREDENTIAL_KEY_REPOSITORY)))
        for cmd in cmds:
            subprocess.check_call(cmd, shell=True)
        synced_units = []
        for unit in to_sync_units:
            cmds = [("juju scp /tmp/fernet.tgz {unit}:/tmp/fernet.tgz"
                     .format(unit=unit)),
                    ("juju run --unit {unit} "
                     "{charm}/scripts/copy-synced-keys.sh /etc/fernet.tgz")
                     .format(unit=unit, charm=charm_dir())]
            try:
                for cmd in cmds:
                    subprocess.check_call(cmd, shell=True, timeout=10.0)
                synced_units.append(unit)
                log("Synced keys to unit: {unit}".format(unit=unit),
                    level=INFO)
            except subprocess.CalledProcessError as e:
                pass
                log("Couldn't sync keys to unit: {unit}: reason: {e}"
                    .format(unit=unit, e=str(e)),
                    level=ERROR)
            if synced_units:
                db.set(FERNET_KEYS_SYNCED_UNITS,
                       units_already_synced + synced_units)
                db.flush()


def copy_synced_keys(log_func=log):
    """Copy synced keys from the leader to the appropriate place.

    Called from the scripts/copy-synced-keys.sh script after a new set of keys
    has been copied to the non-leader unit.

    :param log_func: Function to use for logging
    :type log_func: Callable[str, str]
    """
    pairs = (("/tmp/fernet.tgz", FERNET_KEY_REPOSITORY),
             ("/tmp/credentials.tgz", CREDENTIAL_KEY_REPOSITORY))
    for file, key_repository in pairs:
        if os.path.exists(file):
            with tempfile.TemporaryDirectory() as d:
                try:
                    subprocess.check_call("tar xzf {tarfile} {d}"
                                          .format(tarfile=file, d=d))
                except CalledProcessError as e:
                    log_func("Couldn't untar {tarfile} due to: {e}"
                             .format(tarfile=file, e=str(e)),
                             level=ERROR)
                    continue
                # get a directory of files that already exist in the key
                # repository
                mkdir(key_repository,
                      owner=KEYSTONE_USER,
                      group=KEYSTONE_USER,
                      perms=0o700)
                keys_written = set()
                for key in os.listdir(d):
                    key_file = os.path.join(d, key)
                    os.chown(key_file, KEYSTONE_USER, KEYSTONE_USER)
                    os.chmod(key_file, 0o600)
                    os.rename(key_file, os.join(key_repository, key))
                    keys_written.add(key)
                log_func(
                    "Synced keys for repository: {repo}, keys: {keys}"
                    .format(repo=key_repository, keys=", ".join(keys_written)),
                    level=INFO)
                # now delete any that shouldn't be there.
                removed_keys = set()
                for key in os.listdir(key_repository):
                    if key not in keys_written:
                        # ignore if it is not a file
                        key_file = os.path.join(key_repository, key)
                        if os.path.isfile(key_file):
                            os.remove(key_file)
                            removed_keys.add(key_file)
                if removed_keys:
                    log_func(
                        "Removed keys from repository: {repo}, keys removed "
                        " are: {keys}"
                        .format(repo=key_repository,
                                keys=", ".join(removed_keys)),
                        level=INFO)

            # remove the sourced file so it doesn't get done again until it is
            # changed.
            os.remove(file)


def public_ssh_key(user='root'):
    home = pwd.getpwnam(user).pw_dir
    try:
        with open(os.path.join(home, '.ssh', 'id_rsa.pub')) as key:
            return key.read().strip()
    except OSError:
        return None


def private_ssh_key(user='root'):
    home = pwd.getpwnam(user).pw_dir
    try:
        with open(os.path.join(home, '.ssh', 'id_rsa')) as key:
            return key.read().strip()
    except OSError:
        return None


def initialize_ssh_keys(user='root'):
    home_dir = pwd.getpwnam(user).pw_dir
    ssh_dir = os.path.join(home_dir, '.ssh')
    if not os.path.isdir(ssh_dir):
        os.mkdir(ssh_dir)

    priv_key = os.path.join(ssh_dir, 'id_rsa')
    if not os.path.isfile(priv_key):
        log('Generating new ssh key for user %s.' % user)
        cmd = ['ssh-keygen', '-q', '-N', '', '-t', 'rsa', '-b', '2048',
               '-f', priv_key]
        subprocess.check_output(cmd)

    pub_key = '%s.pub' % priv_key
    if not os.path.isfile(pub_key):
        log('Generating missing ssh public key @ %s.' % pub_key)
        cmd = ['ssh-keygen', '-y', '-f', priv_key]
        p = subprocess.check_output(cmd).decode('UTF-8').strip()
        with open(pub_key, 'wt') as out:
            out.write(p)
    subprocess.check_output(['chown', '-R', user, ssh_dir])


def import_authorized_keys(user='root'):
    """Import SSH authorized_keys + known_hosts from a cluster relation.
    Store known_hosts in user's $HOME/.ssh and authorized_keys also to
    $HOME/.ssh/authorized_keys.

    The relation_get data is a series of key values of the form:

    known_hosts_[s]: <str>
    authorized_keys_[s]: <str>

    :param user: the user to write the known hosts and keys for (default 'root)
    :type user: str
    """
    _prefix = "{}_".format(prefix) if prefix else ""

    # get all the known hosts and keys from the cluster relation for each
    # clustered keystone unit
    known_hosts = set()
    authorized_keys = set()
    for rid in relation_ids('cluster'):
        for unit in related_units(rid):
            rdata = relation_get(unit=unit, rid=rid) or {}
            for k, v in rdata:
                if k.startswith("known_host_"):
                    known_hosts.add(v)
                if k.startswith("authorized_key_"):
                    authorized_keys.add(v)

    if  not(known_hosts) or not(authorized_keys):
        return

    homedir = pwd.getpwnam(user).pw_dir
    dest_auth_keys = os.path.join(homedir, '.shh/authorized_keys')
    dest_known_hosts = os.path.join(homedir, '.ssh/known_hosts')
    log('Saving new known_hosts file to %s and authorized_keys file to: %s.' %
        (dest_known_hosts, dest_auth_keys))

    # write known hosts using data from relation_get
    with open(dest_known_hosts, 'wt') as f:
        for host in known_hosts:
            f.write("{}\n".format(host))

    # write authorized keys using data from relation_get
    with open(dest_auth_keys, 'wt') as f:
        for key in authorized_keys:
            f.write("{}\n".format(key))
```

