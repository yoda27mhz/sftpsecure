#sftp #ssh
## Create container for the lab.
```bash
docker pull  ubuntu
docker run -dit -p 2222:22 --cap-add SYS_ADMIN --security-opt apparmor=unconfined --name testsftp ubuntu
docker ps
docker exec -it testsftp bash
```
- `--cap-add SYS_ADMIN`: Adds the `SYS_ADMIN` capability, which is required for bind mounts.
- `--security-opt apparmor=unconfined`: Disables AppArmor, which can interfere with bind mounts.
## Configuration
```bash
root@d09e835168f0:/# apt-get update
root@d09e835168f0:/# apt-get install openssh-server vim-tiny -y
root@d09e835168f0:/# service ssh start
root@d09e835168f0:/# service --status-all
 [ - ]  dbus
 [ - ]  procps
 [ + ]  ssh
```
Add group:
```bash
root@d09e835168f0:/# groupadd sftpgroup
root@d09e835168f0:/# adduser user1
root@d09e835168f0:/# usermod -aG sftpgroup user1
root@d09e835168f0:/# usermod -s /bin/false user1
root@d09e835168f0:/# adduser user2
root@d09e835168f0:/# usermod -s /bin/false user2
root@d09e835168f0:/# chown root:root /home/user1
root@d09e835168f0:/# chmod 755 /home/user1
root@d09e835168f0:/# chown root:root /home/user2
root@d09e835168f0:/# chmod 755 /home/user2
```
Changing ``usermod -s /bin/false user1`` prevents the user from logging in interactively but allow sftp.
Edit /etc/ssh/sshd_config and add this lines:
```bash
Match Group sftpgroup
        ChrootDirectory %h
        X11Forwarding no
        AllowTcpForwarding no
        PermitTTY no
        ForceCommand internal-sftp
```
or 
```bash
Match User user2
        ChrootDirectory %h
        X11Forwarding no
        AllowTcpForwarding no
        PermitTTY no
        ForceCommand internal-sftp
```
- **ChrootDirectory %h** chroot to home.
- **X11Forwarding no** To disable X11 forwarding in the SSH server
- **AllowTcpForwarding no** disables TCP forwarding for all users by `Match` directive.
- **ForceCommand internal-sftp** forces all SSH connections to use the SFTP protocol only
- **PermitTTY no** in the SSH configuration file (`sshd_config`) prevents users from accessing a terminal session, which is useful for restricting users to SFTP access only.
Getting this error:
```bash
PS C:\Users\raperezs> ssh -p 2222 user1@192.168.1.2
user1@192.168.1.2's password:
PTY allocation request failed on channel 0
This service allows sftp connections only.
Connection to 192.168.1.2 closed.
```
## Restart the service
Check if `sshd_config` have any syntax error.
```bash
root@d09e835168f0:/# sshd -t
root@d09e835168f0:/# service ssh restart
 * Restarting OpenBSD Secure Shell server sshd                                                                   [ OK ]
```
## Access to another directory from another path.
Using a bind mount in SFTP configurations allows you to map a directory from another part of the filesystem into the chroot environment. This is particularly useful when you want to provide access to files outside the chroot directory while maintaining security and isolation. Here are the steps and considerations for setting up a bind mount for SFTP:
Example:
```bash
root@d09e835168f0:/# mkdir -p /projects/project{1..3}
root@d09e835168f0:/# mkdir -p /home/user1/project{1..2}
root@d09e835168f0:/# mkdir -p /home/user2/project3
root@d09e835168f0:/# mount -o bind /projects/project1 /home/user1/project1
root@d09e835168f0:/# mount -o bind /projects/project2 /home/user1/project2
root@d09e835168f0:/# mount -o bind /projects/project3 /home/user2/project3
root@d09e835168f0:/# mkdir -p /projects/project{1..3}/DOWNLOAD
root@d09e835168f0:/# mkdir -p /projects/project{1..3}/UPLOAD
root@d09e835168f0:/# chmod 777 /projects/project{1..3}/UPLOAD
```
For UPLOAD should have properly rights to upload files. 
- **Mounting a Directory**: To bind mount a directory, you use the `mount --bind` command. For example, if you want to mount `/mnt/data/project1` into `/home/user1/project1`, you would run:
    ```
    mount -o bind /mnt/data/project1 /home/user1/project1
    ```
    This command maps the `/mnt/data/project1` directory into the `/home/user1/project1` directory within the chroot environment.
- **Persistent Mounts**: To ensure the bind mount persists after a reboot, you need to add an entry to the `/etc/fstab` file. For example:
    ```
    /mnt/data/project1 /home/user1/project1 none bind 0 0
    ```
    This line in `/etc/fstab` ensures that the directory is mounted at boot time.
- **Permissions**: The bind mount path needs to be fully owned by root, but files and subdirectories do not necessarily have to be. However, SSHD is strict about permissions. The top-level directory of the chroot environment must be owned by root and not writable by any other user or group. For example:
    ```
    chown root:root /home/user1
    chmod 755 /home/user1
    ```
- **Security Considerations**: While bind mounts can be convenient, they can also introduce security risks if not managed properly. For instance, if the mounted directory is not owned by root or is writable by other users, it can bypass the security checks imposed by SSHD. Therefore, it’s important to ensure that the mounted directory and its contents adhere to the strict permission requirements.
- **Alternative Methods**: Instead of using bind mounts, you can also move the document root directory to the user’s home directory and then symlink it to the original location. This avoids the need for bind mounts but requires moving files around.
By following these steps and considerations, you can effectively use bind mounts to provide SFTP access to directories outside the chroot environment while maintaining security and system integrity.
