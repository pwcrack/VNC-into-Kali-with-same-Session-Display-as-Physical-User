# VNC INTO KALI WITH THE SAME SESSION/DISPLAY AS THE PHYSICAL USER

If you run VNC as service (e.g. x11vnc) on Linux and connect it to Display :0 (the Physical Display) and make it available on port 5900 (the default port), everything will work fine until the screen is locked at which point the VNC Connection will go Black.  Below is a detailed explanation and instructions for installing and using VNC to connect to the actual physical session and display of the physical user on debian linux.  I did this using Kali Linux 2021.4a, but this should work on a wide variety of Debian distros include Ubuntu, etc.  Kali Linux uses LightDM as its Display Manager and for part of these instructions I make use of LightDM functionality.  Therefore, as a pre-requisite, if you are not using LightDM, you may have to slightly modify these instructions for your Display Manager (Gnome Display Manager/GDM, etc.) or change your Display Manager to LightDM (there are instructions for doing that elsewhere on the internet.)  I am using x11vnc as my vncserver.  Xvnc, TightVNC, TIgerVNC and other VNC servers should also work as long as they offer the capability to specify a display (i.e. :0 or :1).
This is not totally "a solution.”  It is a bit of a work-around because it will require you to VNC into two different ports to complete the normal unlock process.

# THE PROBLEM:

Why is this even a problem?  When you VNC into a Windows machine (assuming a VNC server has been deployed there), and lock the session, VNC remains available on port 5900.  You can still see the GUI login screen/prompt on VNC just as you would if you were sitting at the physical machine.  When you login, you connect to the physical display without an issue.  However, linux sessions, seats and displays do not work the same way as Windows.  There are lots of other pages of instructions on the internet that will show you how to create a VNC server on port 5900 for a new session.  Connecting to this port will offer you a new GUI login and new session, but you will not see the active desktop display of the physical user (display :0) the way you do on Windows.  Successive connections will create new sessions.  Other instructions will allow you to create a VNC connection to display :0 and with these you will see the display of the physical user, until the session is locked.  Then the screen goes solid black and there is no way to login.  What happened?  How can we make Linux act like Windows (at least as much as possible) so we can VNC connect to the physical display and continue to do so after session locks.  When Debian boots, the Display Manager (for instance LightDM) creates a greeter (GUI login screen) and connects it to Display :0.  When you login, a User Session is created and this replaces the connection to Display :0.  However, when the user session is locked, a new greeter (GUI login screen) is now connected to Display :1 and Display :0 is Blacked out.

After fresh reboot, LightDM GUI login screen is available on Display :0.  The LightDM user session in this example is: c12.  Session 23 is the ssh connection I am using to show these commands (there is no display attached to the ssh session, but you can test that by running the same command and giving the ssh session ID.)

![Screenshot 1](/Capture1.PNG?raw=true "Screenshot 1")

Assuming we have set-up a VNC service, we can VNC connect on port 5900 and we will see the login screen and be able to login to the physical desktop display.  We now have user session (ID = 30) with Display :0 and this display will still be available on port 5900 (so we will see this in our VNC connection.)

![Screenshot 2](/Capture2.PNG?raw=true "Screenshot 2")

However, If we lock the session through VNC or on the physical desktop, the VNC connection will go to a Black screen.  We can disconnect it (but it doesn’t automatically disconnect or drop the connection.)  Display :0 is still there and we still have a VNC service connected to it, but we can’t see anything because of the Display Manager (LightDM.)  If we were to reconnect VNC to port 5900, we would still see the Black screen.  The Display Manager (LightDM) has spawned a new session (c13) on Display :1 for the new login GUI.  We can see this below and our goal will be to connect to this session/display.

![Screenshot 3](/Capture3.PNG?raw=true "Screenshot 3")

# THE WORKAROUND:

Assuming we already have a VNC service for Display :0 available on port 5900, let’s make a second VNC service connected to Display :1 and available on port 5901.  If the machine has been freshly rebooted, we can VNC connect to 5900, login and then work the desktop remotely.  However, if we lock the desktop, the screen will go Black.  To log back in we need to connect to port 5901.  However, there is a second issue.  When LightDM creates the new Session for the login gui on Display :1, the service (which we will enable to run at boot) doesn’t recognize it.  We need to restart the service each time this happens.  We can do this by adding a line to the [Seat:*] section of the LightDM configuration.  With this done, we’ll be able to VNC into port 5901 and unlock the locked physical session on the desktop.  When we do this the LightDM Display Manager will make the physical desktop display available again on port 5900 and we can VNC back into it on that port.  We will lose our VNC connection on port 5901, but we can reconnect on port 5900 and see our gui physical desktop on display :0.  Actions taken by a physical user sitting at the machine will be analogous and will not affect the VNC set-up.  I am going to provide detailed instructions on how to set up a VNC password (separate from the user login) and to tunnel over SSH for additional security.  Typically, my machines are local only IP addresses and I VNC from other local machines on the same network.  When I am physically remote, I VPN into the local network.  In that situation I would be tunneling password-protected (but unencrypted) VNC over SSH and tunneling that over a VPN.
# PUTTING IT TOGETHER STEP-BY-STEP:

1.	Install x11vnc

sudo apt install x11vnc


2.	Create your VNC password

x11vnc -storepasswd

3.	Test your VNC connection.  On your linux machine run

x11vnc -create -xkb -display :0 -noxrecord -noxfixes -noxdamage -rfbauth /home/yourusername/.vnc/passwd -usepw

From a second machine connect to your linux machine using VNC on port 5900.  For instance:

vncviewer 192.168.1.2:5900

You will be prompted for the VNC password.  You should then see your physical desktop.  Note that because this is just a testing connection, this connection will not persist after the first connection is disconnected.  You would need to re-run the command again to test a second time.

4.	Make this a service (note that the ExecStart command is a little bit different from above.  We have added -forever so that VNC will remain running after a disconnection.)

Using sudo create this file:  /etc/systemd/system/x11vnc.service with the following content:

[Unit]<br/>
Description="x11vnc"<br/>
Requires=display-manager.service<br/>
After=display-manager.service<br/>
<br/>
[Service]<br/>
ExecStart=/usr/bin/x11vnc -create -xkb -display :0 -noxrecord -noxfixes -noxdamage -o /var/log/x11vnc.log -rfbauth /home/diskun/.vnc/passwd -usepw -auth guess -forever<br/>
ExecStop=/usr/bin/killall x11vnc<br/>
Restart=on-failure<br/>
Restart-sec=200<br/>
<br/>
[Install]<br/>
WantedBy=multi-user.target<br/>

5.	Make a second service (note this service is a little bit different from the first.  The filename is different, the Description is different and the ExecStart command now includes “display :1 -rfbport 5901” to make display :1 available as a service on port 5901.

Using sudo create this second file (note the filename change):  /etc/systemd/system/x11vnc1.service with the following content:

[Unit]<br/>
Description="x11vnc1"<br/>
Requires=display-manager.service<br/>
After=display-manager.service<br/>
<br/>
[Service]<br/>
ExecStart=/usr/bin/x11vnc -create -xkb -display :1 -rfbport 5901 -noxrecord -noxfixes -noxdamage -o /var/log/x11vnc.log -rfbauth /home/diskun/.vnc/passwd -usepw -auth guess -forever<br/>
ExecStop=/usr/bin/killall x11vnc<br/>
Restart=on-failure<br/>
Restart-sec=200<br/>
<br/>
[Install]<br/>
WantedBy=multi-user.target<br/>

6.	Start and enable both services

sudo systemctl daemon-reload<br/>
sudo systemctl startx11vnc<br/>
sudo systemctl statusx11vnc<br/>
sudo systemctl startx11vnc1<br/>
sudo systemctl status x11vnc1<br/>
sudo systemctl enable x11vnc<br/>
sudo systemctl enable x11vnc1<br/>

With your system still unlocked, from a second system you can test your set-up again by connecting over VNC on port 5900 and you should still be prompted for your VNC password and you should then be able to see your physical desktop.

7.	Modify LightDM Configuration

sudo nano /etc/lightdm/lightdm.conf

in the [Seat:*] section find the line that currently reads:  “#display-setup-script=” and uncomment it and change it to:

display-setup-script=sudo service x11vnc restart

8.	Restart LightDM

Be careful with this next command.  This is like a reboot and will close all your open windows and force a login.  Make sure to save any open work before you run this command.

sudo service lightdm restart

You should now be ready to test your remote VNC connection.  Since you just restarted lightdm, from a remote machine you can connect to VNC on port 5900.  You should see your physical login screen.  As you login over VNC you should be able to see the same login on the physical machine.  If you lock your session (in VNC or physically) the VNC screen should go to Black.  When this happens disconnect from VNC.  To restore your VNC session, create a VNC connection to your machine on port 5901.  You will be prompted for you VNC password.  You will then see the physical login screen.  Login and as you complete the login VNC will disconnect.  The physical machine will have its desktop unlocked and you can now connect to the desktop through VNC on port 5900 again.

To tunnel VNC over ssh, from linux first create the tunnels:
 
ssh -L 5900:localhost:5900 -L 5901:localhost:5901 USER@REMOTE_IP

You will be prompted for your ssh login password.  Then VNC connect to localhost:5900 or localhost:5901 with your remote VNC Viewer (depending upon whether this is the first login or your are unocking a previously locked session.)  You will be prompted for you VNC password.

On Windows, in PuTTY, create an SSH session with the remote IP to your linux box on port 22 and then in the Connection category, expand SSH and click on Tunnels.  Add the following two tunnels to your SSH connection (changing remoteIP to the IP address of the remote box you are connecting to.)  You will be prompted for your SSH login.  Once connected, VNC to localhost:5900 or localhost:5901 as appropriate, depending upon whether you have just rebooted or you are unlocking a previously locked session.  You will be prompted for your VNC password.

![Screenshot 4](/Capture4.PNG?raw=true "Screenshot 4")
