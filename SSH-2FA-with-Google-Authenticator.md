Adding Google Authenticator to SSH logins
===

## Prerequisites

- Debian / Ubuntu based Linux
- OpenSSH setup and working
- Existing access to the server's console
- Root priveledges to setup initially; user access later

## Purpose
This will let us use our phone as a second-factor for logging in to an SSH session. This helps harden our SSH security, and encourages users to use 2FA in general. Because it asks for a password that changes every 60 seconds along with your known password, it makes the server MUCH harder to brute-force, especially if it is an Internet-facing server.

## Directions

1. Update your apt repos, and install the required packages:
```
sudo apt update
sudo apt install ssh libpam-google-authenticator
```

2. Ensure you have a spare SSH session open, as the following steps *could lock you out remotely*. You have been warned.
3. Edit your `/etc/pam.d/sshd` file, and add the following stanza to the very end:
```
# Google Authenticator
auth required pam_unix.so try_first_pass
auth required pam_google_authenticator.so secret=${HOME}/.google_authenticator
```
> At this point, users cannot log in anymore.

4. If you only want this on certain accounts, create a stanza in `/etc/ssh/sshd_config` to `Match User`:
```
Match User usernameGoesHere
    AuthenticationMethods publickey keyboard-interactive:pam
```

In the above, `usernameGoesHere` can either log in with a Public Key, or with their password and 2-factor code (which is next to be setup). If you want it to match everyone, remove the `Match User ...` line.

5. Ensure that the following lines are in your `/etc/ssh/sshd_config` file:
```
ChallengeResponseAuthentication yes
UsePAM yes
```

6. As the user, run `google-authenticator`, and follow the directions. Choose `Y` for asking if you want tokens to be time-based. It will output a QR code that you can scan with your device. It will also save a file to the home directory named `.google_authenticator`.

> **Save this file somewhere securely**. If you drop this file into another user's home directory, they will use the same OTP as you. This is great for backing up and restoring as well, or using the same OTP on multiple servers under your control.

Ensure the file has only Read permissions for the owner (`sudo chmod 0600 ${HOME}/.google_authenticator`), otherwise it will fail to read it for security purposes. The standard user *should not modify this file manually*. If they want a new set of codes, the user will need to re-run the `google-authenticator` command to generate a new QR code, recovery codes, etc.

7. Once this is done, the moment of truth... Restart the SSH service, and attempt to log in with your second console!
```
sudo systemctl restart ssh.service
```

Ideally, it should ask you to log in with your password, then ask for a verification code (whether the password was right or wrong). If you are successful, it should log you in. If not, use your still-open SSH session to fix it.

# Recovery Codes

Running the `google-authenticator` command successfully will also output your `Emergency scratch codes`, which are useful if you lose access to your mobile device. You can only use these codes once. You'll notice they are 8 digits long, instead of the standard 6. **Keep this safe and not on your phone**. There is no sense having your backup on your primary use device.

If you ever need to view the recovery codes (i.e. your piece of paper was ripped to shreds and thrown in a fire), you can `cat ${HOME}/.google_authenticator` to view them. You'll also see the Secret Key as the first line of the file. Using this Secret Key, you can add another authenticator to use the same code by manually typing this in (for example, sharing an account with a friend for some weird, unknown reason).