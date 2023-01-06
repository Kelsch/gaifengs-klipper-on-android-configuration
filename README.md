# gaifengs-klipper-on-android-configuration
I didn't know how to create a branch or whatever so, I created my own repository. This is 100% just gaifengs english readme with some changes to make it easier to get klipper running through octo4a.

Please go to https://github.com/gaifeng8864/klipper-on-android/blob/main/README_en.md he is the true mastermind behing all of this!

There is a pretty high chance that someone with more knowledge will be able to make this easier soon and I hope they do. Or even a chance that this doesn't even work, but somehow worked for me.

I really don't know bash scripts.


#!/bin/bash

if [ "$(id  -u)" = "0" ]
then
  echo "Start script as user!"
  exit 1
fi

if ! [[ -e "./klippy-env" && -e "./moonraker-env" && -e "./fluidd" && -e "./.KlipperScreen-env" ]]
then
  echo "First use kiauh to install the klipper family!"
  exit 1
fi

if [ ! -c "/dev/pts/0" ] || [ ! -c "/dev/pts/1" ] || [ ! -c "/dev/pts/4" ] 
then
  echo "Please connect your phone to the printer "
  exit 1
fi

### environment
echo "Initializing environment variables"

KLIPPER_USER=$USER

ETC_DEFAULT_KLIPPER=/etc/default/klipper
ETC_DEFAULT_MOONRAKER=/etc/default/moonraker

ETC_INIT_KLIPPER=/etc/init.d/klipper
ETC_INIT_MOONRAKER=/etc/init.d/moonraker

USR_LOCAL_BIN_XTERM=/usr/local/bin/xterm

TTYFIX="/usr/bin/ttyfix"
TTYFIX_START="/etc/init.d/ttyfix"

POWERFIX="/usr/bin/powerfix"
POWERFIX_START="/etc/init.d/powerfix"

### packages
echo "Installing required packages"

sudo apt install -y inotify-tools fonts-wqy-zenhei iw

### Configuration for power
sudo tee "$POWERFIX" <<EOF
#!/bin/bash
sudo unchroot dumpsys battery set status 2
sudo unchroot dumpsys battery set level 98
sudo unchroot iw wlan0 set power_save off 
EOF
sudo chmod +x "$POWERFIX"

sudo tee "$POWERFIX_START" <<EOF
#!/bin/sh
### BEGIN INIT INFO
# Provides:          powerfix
# Default-Start:        2 3 4 5
# Default-Stop:
# Required-Start:    \$local_fs \$remote_fs
# Short-Description: powerfix
# Description: powerfix
### END INIT INFO

$POWERFIX

exit 0
EOF
sudo chmod +x "$POWERFIX_START"

### Configuration for Klipperscreen_xterm
sudo tee "$USR_LOCAL_BIN_XTERM" <<EOF
#!/bin/bash
# Configuration for Klipperscreen_xterm

KLIPPERSCREEN_CONFIG=/home/$KLIPPER_USER/KlipperScreen.conf
KLIPPERSCREEN_LOG=/home/$KLIPPER_USER/printer_data/logs/KlipperScreen.log
sudo unchroot am start -n x.org.server/x.org.server.MainActivity >/dev/null 2>&1
sleep 10
/home/$KLIPPER_USER/.KlipperScreen-env/bin/python /home/$KLIPPER_USER/KlipperScreen/screen.py -c \$KLIPPERSCREEN_CONFIG -l \$KLIPPERSCREEN_LOG
EOF
sudo chmod +x $USR_LOCAL_BIN_XTERM

### Configuration for pts
sudo tee "$TTYFIX" <<EOF
#!/bin/bash

inotifywait -m /dev -e create |
  while read dir action file
  do
    ([ "\$file" = "0" ] && chmod 777 /dev/pts/0) || ([ "\$file" = "1" ] && chmod 777 /dev/pts/1) || ([ "\$file" = "4" ] && chmod 777 /dev/pts/4)
  done
EOF
sudo chmod +x "$TTYFIX"

sudo tee "$TTYFIX_START" <<EOF
#!/bin/sh
### BEGIN INIT INFO
# Provides:          ttyfix
# Default-Start:        2 3 4 5
# Default-Stop:
# Required-Start:    \$local_fs \$remote_fs
# Short-Description: ttyfix
# Description: ttyfix
### END INIT INFO

. /lib/lsb/init-functions

N="$TTYFIX_START"
PIDFILE=/run/ttyfix.pid
EXEC="$TTYFIX"

set -e

f_start ()
{
  start-stop-daemon --start --background --make-pidfile --pidfile \$PIDFILE --exec \$EXEC
}

f_stop ()
{
  start-stop-daemon --stop --pidfile \$PIDFILE
}

case "\$1" in
  start)
        f_start
        ;;
  stop)
        f_stop
        ;;
  restart)
        f_stop
        sleep 1
        f_start
        ;;
  reload|force-reload|status)
        ;;
  *)
        echo "Usage: \$N {start|stop|restart|force-reload|status}" >&2
        exit 1
        ;;
esac

exit 0
EOF
sudo chmod +x "$TTYFIX_START"

### Configuration for /etc/init.d/klipper
sudo tee "$ETC_DEFAULT_KLIPPER" <<EOF
KLIPPY_CONFIG="/home/$KLIPPER_USER/printer_data/config/printer.cfg"
KLIPPY_LOG="/home/$KLIPPER_USER/printer_data/logs/klippy.log"
KLIPPY_SOCKET="/home/$KLIPPER_USER/printer_data/comms/klippy.sock"
KLIPPY_PRINTER=/tmp/printer
KLIPPY_EXEC="/home/$KLIPPER_USER/klippy-env/bin/python"
KLIPPY_ARGS="/home/$KLIPPER_USER/klipper/klippy/klippy.py \$KLIPPY_CONFIG -l \$KLIPPY_LOG -a \$KLIPPY_SOCKET"
EOF

### Configuration for /etc/init.d/moonraker
sudo tee "$ETC_DEFAULT_MOONRAKER" <<EOF
MOONRAKER_CONFIG="/home/$KLIPPER_USER/printer_data/config/moonraker.conf"
MOONRAKER_LOG="/home/$KLIPPER_USER/printer_data/logs/moonraker.log"
MOONRAKER_SOCKET=/tmp/moonraker_uds
MOONRAKER_PRINTER=/tmp/printer
MOONRAKER_EXEC="/home/$KLIPPER_USER/moonraker-env/bin/python"
MOONRAKER_ARGS="/home/$KLIPPER_USER/moonraker/moonraker/moonraker.py -c \$MOONRAKER_CONFIG -l \$MOONRAKER_LOG"
EOF

### System startup script for Klipper 3d-printer host code
sudo tee "$ETC_INIT_KLIPPER" <<EOF
#!/bin/sh
# System startup script for Klipper 3d-printer host code

### BEGIN INIT INFO
# Provides:          klipper
# Required-Start:    \$local_fs
# Required-Stop:
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Short-Description: Klipper daemon
# Description:       Starts the Klipper daemon.
### END INIT INFO

PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
DESC="klipper daemon"
NAME="klipper"
DEFAULTS_FILE=/etc/default/klipper
PIDFILE=/var/run/klipper.pid

. /lib/lsb/init-functions

# Read defaults file
[ -r \$DEFAULTS_FILE ] && . \$DEFAULTS_FILE

case "\$1" in
start)  chmod 777 /dev/pts/0
        chmod 777 /dev/pts/1
        chmod 777 /dev/pts/4
        log_daemon_msg "Starting" \$NAME
        start-stop-daemon --start --quiet --exec \$KLIPPY_EXEC \\
		                  --background --pidfile \$PIDFILE --make-pidfile \\
		                  --chuid $KLIPPER_USER --user $KLIPPER_USER \\
		                  -- \$KLIPPY_ARGS
        log_end_msg \$?
        ;;
stop)   log_daemon_msg "Stopping" \$NAME
        killproc -p \$PIDFILE \$KLIPPY_EXEC
        RETVAL=\$?
        [ \$RETVAL -eq 0 ] && [ -e "\$PIDFILE" ] && rm -f \$PIDFILE
        log_end_msg \$RETVAL
        ;;
restart) log_daemon_msg "Restarting" \$NAME
        \$0 stop
        \$0 start
        ;;
reload|force-reload)
        log_daemon_msg "Reloading configuration not supported" \$NAME
        log_end_msg 1
        ;;
status)
        status_of_proc -p \$PIDFILE \$KLIPPY_EXEC \$NAME && exit 0 || exit \$?
        ;;
*)      log_action_msg "Usage: /etc/init.d/klipper {start|stop|status|restart|reload|force-reload}"
        exit 2
        ;;
esac
exit 0
EOF
sudo chmod +x "$ETC_INIT_KLIPPER"

### System startup script for Moonraker API for Klipper
sudo tee "$ETC_INIT_MOONRAKER" <<EOF
#!/bin/sh
# System startup script for Moonraker API for Klipper

### BEGIN INIT INFO
# Provides:          moonraker
# Required-Start:    \$local_fs \$remote_fs klipper
# Required-Stop:
# Default-Start:     2 3 4 5
# Default-Stop:
# Short-Description: Moonraker daemon
# Description:       Starts the Moonraker daemon.
### END INIT INFO

PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
DESC="moonraker daemon"
NAME="moonraker"
DEFAULTS_FILE=/etc/default/moonraker
PIDFILE=/var/run/moonraker.pid

. /lib/lsb/init-functions

# Read defaults file
[ -r \$DEFAULTS_FILE ] && . \$DEFAULTS_FILE

case "\$1" in
start)  log_daemon_msg "Starting" \$NAME
        start-stop-daemon --start --quiet --exec \$MOONRAKER_EXEC \\
                          --background --pidfile \$PIDFILE --make-pidfile \\
                          --chuid $KLIPPER_USER --user $KLIPPER_USER \\
                          -- \$MOONRAKER_ARGS
        log_end_msg \$?
        ;;
stop)   log_daemon_msg "Stopping" \$NAME
        killproc -p \$PIDFILE \$MOONRAKER_EXEC
        RETVAL=\$?
        [ \$RETVAL -eq 0 ] && [ -e "\$PIDFILE" ] && rm -f \$PIDFILE
        log_end_msg \$RETVAL
        ;;
restart) log_daemon_msg "Restarting" \$NAME
        \$0 stop
        \$0 start
        ;;
reload|force-reload)
        log_daemon_msg "Reloading configuration not supported" \$NAME
        log_end_msg 1
        ;;
status)
        status_of_proc -p \$PIDFILE \$MOONRAKER_EXEC \$NAME && exit 0 || exit \$?
        ;;
*)      log_action_msg "Usage: /etc/init.d/moonraker {start|stop|status|restart|reload|force-reload}"
        exit 2
        ;;
esac
exit 0
EOF
sudo chmod +x $ETC_INIT_MOONRAKER

### Configure autostart service
sudo update-rc.d ttyfix defaults 
sudo update-rc.d klipper defaults 
sudo update-rc.d moonraker defaults 
sudo update-rc.d powerfix defaults

### complete
echo "Configuration complete , Please restart your phone!!!"








