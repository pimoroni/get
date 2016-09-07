#!/bin/bash

: <<'DISCLAIMER'

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND,
EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES
OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND
NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT
HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY,
WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING
FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR
OTHER DEALINGS IN THE SOFTWARE.

This script is licensed under the terms of the MIT license.
Unless otherwise noted, code reproduced herein
was written for this script.

- The Pimoroni Crew -

DISCLAIMER

# script control variables

productname="Unicorn HAT/pHAT" # the name of the product to install
scriptname="unicornhat" # the name of this script
debugmode="no" # whether the script should use debug routines
debuguser="none" # optional test git user to use in debug mode
debugpoint="none" # optional git repo branch or tag to checkout
forcesudo="no" # whether the script requires to be ran with root privileges
promptreboot="no" # whether the script should always prompt user to reboot
customcmd="yes" # whether to execute commands specified before exit
i2creq="no" # whether i2c is required or not
spireq="no" # whether spi is required or not
uartreq="no" # whether uart communication is required or not
armhfonly="yes" # whether the script is allowed to run on other arch
armv6="yes" # whether armv6 processors is supported or not
armv7="yes" # whether armv7 processors is supported or not
armv8="yes" # whether armv8 processors is supported or not
raspbianonly="yes" # whether the script is allowed to run on other OSes
macosxsupport="no" # whether Mac OS X is supported by the script
osreleases=( "Raspbian" "Ubuntu" "Debian" ) # list all the os-releases supported
squeezesupport="no" # whether Squeeze is supported or not
wheezysupport="yes" # whether Wheezy is supported or not
jessiesupport="yes" # whether Jessie is supported or not
debpackage="na" # the name of the package in apt repo
piplibname="unicornhat" # the name of the lib in pip repo
pipoverride="yes" # whether the script should give priority to pip repo
pip2support="yes" # whether python2 is supported or not
pip3support="yes" # whether python3 is supported or not
topdir="Pimoroni" # the name of the top level directory
localdir="unicornhat" # the name of the dir for copy of examples
gitreponame="unicorn-hat" # the name of the git project repo
gitusername="pimoroni" # the name of the git user to fetch repo from
gitrepobranch="master" # repo branch to checkout
gitrepotop="root" # the name of the dir to base repo from
gitrepoclone="no" # whether the git repo is to be cloned locally
gitclonedir="source" # the name of the local dir for repo
repoclean="no"  # whether any git repo clone found should be cleaned up
repoinstall="no" # whether the library should be installed from repo
libdir="library" # subdirectory of library in repo
copydir=( "documentation" "examples" ) # subdirectories to copy from repo
pipdeplist=() # list of dependencies to source from pip
pkgaptremove=() # list of conflicting apt packages to remove
pkgdeplist1=() # list core dependencies
pkgdeplist2=( "python-pip" "python-dev" ) # list python 2 dependencies
pkgdeplist3=( "python3-pip" "python3-dev" ) # list python 3 dependencies
pkgdeplist7=() # list dependencies for Wheezy only
pkgdeplist8=() # list dependencies for Jessie only
moreaptdep=( "git" "python-numpy" "python3-numpy" "python-flask" "python3-flask" ) # list of all the additional apt dependencies
morepipdep=() # list of all the additional pip dependencies
spacereq=150 # minimum size required on root partition in MB

# template 1609071650

FORCE=$1
ASK_TO_REBOOT=false
CURRENT_SETTING=false
MIN_INSTALL=false
FAILED_PKG=false
REMOVE_PKG=false
UPDATE_DB=false

BOOTCMD=/boot/cmdline.txt
CONFIG=/boot/config.txt
APTSRC=/etc/apt/sources.list
INITABCONF=/etc/inittab
BLACKLIST=/etc/modprobe.d/raspi-blacklist.conf
LOADMOD=/etc/modules

RASPOOL="http://mirrordirector.raspbian.org/raspbian/pool"
DEBPOOL="http://ftp.debian.org/debian/pool"

# function define

confirm() {
    if [ "$FORCE" == '-y' ]; then
        true
    else
        read -r -p "$1 [y/N] " response < /dev/tty
        if [[ $response =~ ^(yes|y|Y)$ ]]; then
            true
        else
            false
        fi
    fi
}

prompt() {
        read -r -p "$1 [y/N] " response < /dev/tty
        if [[ $response =~ ^(yes|y|Y)$ ]]; then
            true
        else
            false
        fi
}

success() {
    echo "$(tput setaf 2)$1$(tput sgr0)"
}

warning() {
    echo "$(tput setaf 1)$1$(tput sgr0)"
}

newline() {
    echo ""
}

sudocheck() {
    if [ $(id -u) -ne 0 ]; then
        echo -e "Install must be run as root. Try 'sudo ./$scriptname'\n"
        exit 1
    fi
}

sysclean() {
    sudo apt-get clean && sudo apt-get autoclean
    sudo apt-get -y autoremove &> /dev/null
}

sysupdate() {
    if ! $UPDATE_DB; then
        sudo apt-get update
        UPDATE_DB=true
    fi
}

sysupgrade() {
    sudo apt-get update && sudo apt-get upgrade
    sudo apt-get clean && sudo apt-get autoclean
    sudo apt-get -y autoremove &> /dev/null
}

sysreboot() {
    warning "Some changes made to your system require"
    warning "your computer to reboot to take effect."
    newline
    if prompt "Would you like to reboot now?"; then
        sync
        sudo reboot
    fi
}

arch_check() {
    IS_ARMHF=false
    IS_ARMv6=false

    if uname -m | grep "armv.l" > /dev/null; then
        IS_ARMHF=true
        if uname -m | grep "armv6l" > /dev/null; then
            IS_ARMv6=true
        fi
    fi
}

os_check() {
    IS_RASPBIAN=false
    IS_MACOSX=false
    IS_SUPPORTED=false

    if [ -f /etc/os-release ]; then
        if cat /etc/os-release | grep "Raspbian" > /dev/null; then
            IS_RASPBIAN=true && IS_SUPPORTED=true
        fi
        if command -v apt-get > /dev/null; then
            for os in ${osreleases[@]}; do
                if cat /etc/os-release | grep "$os" > /dev/null; then
                    IS_SUPPORTED=true
                fi
            done
        fi
    elif uname -s | grep "Darwin" > /dev/null; then
        IS_MACOSX=true
        if [ $macosxsupport == "yes" ]; then
            IS_SUPPORTED=true
        fi
    fi
}

raspbian_check() {
    IS_SQUEEZE=false
    IS_WHEEZY=false
    IS_JESSIE=false

    if [ -f /etc/os-release ]; then
        if cat /etc/os-release | grep "jessie" > /dev/null; then
            IS_JESSIE=true
        elif cat /etc/os-release | grep "wheezy" > /dev/null; then
            IS_WHEEZY=true
        elif cat /etc/os-release | grep "squeeze" > /dev/null; then
            IS_SQUEEZE=true
        else
            echo "Unsupported distribution"
            exit 1
        fi
    fi
}

raspbian_old() {
    if $IS_SQUEEZE || $IS_WHEEZY ;then
        true
    else
        false
    fi
}

home_dir() {
    if ! $IS_MACOSX; then
        if [ $EUID -ne 0 ]; then
            USER_HOME=$(getent passwd $USER | cut -d: -f6)
        else
            if [ $SUDO_USER ]; then
                USER_HOME=$(getent passwd $SUDO_USER | cut -d: -f6)
            else
                warning "Running as root and no other sudo user available"
                exit 1
            fi
        fi
    else
        if [ $EUID -ne 0 ]; then
            USER_HOME=$(dscl . -read /Users/$USER NFSHomeDirectory | cut -d: -f2)
        else
            USER_HOME=$(dscl . -read /Users/$SUDO_USER NFSHomeDirectory | cut -d: -f2)
        fi
    fi
}

servd_trig() {
    if command -v service > /dev/null; then
        sudo service $1 $2
    fi
}

servd_trig() {
    if command -v service > /dev/null; then
        sudo service $1 $2
    fi
}

get_init_sys() {
    if command -v systemctl > /dev/null && systemctl | grep -q '\-\.mount'; then
        SYSTEMD=1
    elif [ -f /etc/init.d/cron ] && [ ! -h /etc/init.d/cron ]; then
        SYSTEMD=0
    else
        echo "Unrecognised init system" && exit 1
    fi
}

i2c_vc_dtparam() {
    if [ -e $CONFIG ] && grep -q "^dtparam=i2c_vc=on$" $CONFIG; then
        newline
        echo "i2c0 bus already active"
    else
        newline
        echo "Enabling i2c0 bus in $CONFIG"
        echo "dtparam=i2c_vc=on" | sudo tee -a $CONFIG
        newline
    fi
}

usb_max_power() {
    if [ -e $CONFIG ] && grep -q "^max_usb_current=1$" $CONFIG; then
        newline
        echo "Max USB current setting already active"
    else
        newline
        echo "Adjusting USB current setting in $CONFIG"
        echo "max_usb_current=1" | sudo tee -a $CONFIG
        newline
    fi
}

force_hdmi() {
    if [ -e $CONFIG ] && grep -q "^hdmi_force_hotplug=1$" $CONFIG; then
        \curl -sS https://get.pimoroni.com/audio | sudo bash -s - "2"
    else
        echo "hdmi_force_hotplug=1" | sudo tee -a $CONFIG &> /dev/null
        \curl -sS https://get.pimoroni.com/audio | sudo bash -s - "2"
        ASK_TO_REBOOT=true
    fi
}

space_chk() {
    if command -v stat > /dev/null && ! $IS_MACOSX; then
        if [ $spacereq -gt $(($(stat -f -c "%a*%S" /)/10**6)) ];then
            newline
            warning  "There is not enough space left to proceed with  installation"
            if confirm "Would you like to attempt to expand your filesystem?"; then
                \curl -sS https://get.pimoroni.com/expandfs | sudo bash
                exit 1
            else
                newline
                exit 1
            fi
        fi
    fi
}

check_network() {
    sudo ping -q -w 1 -c 1 `ip r | grep default | cut -d ' ' -f 3` &> /dev/null && return 0 || return 1
}

apt_pkg_req() {
    APT_CHK=$(dpkg-query -W -f='${Status}\n' $1 2> /dev/null | grep "install ok installed")

    if [ "" == "$APT_CHK" ]; then
        true
    else
        false
    fi
}

apt_pkg_install() {
    \curl -sS https://get.pimoroni.com/package | sudo bash -s - $1 || { warning "Apt failed to install $1!" && FAILED_PKG=true; }
    newline
}

pip2_chk() {
    if command -v pip2 > /dev/null; then
        PIP2_BIN="pip2"
    elif command -v pip-2.7 > /dev/null; then
        PIP2_BIN="pip-2.7"
    elif command -v pip-2.6 > /dev/null; then
        PIP2_BIN="pip-2.6"
    else
        PIP2_BIN="pip"
    fi
}

pip2_pkg_req() {
    PIP2_CHK=$($PIP2_BIN search $1 | grep INSTALLED)

    if [ "" == "$PIP2_CHK" ]; then
        true
    else
        false
    fi
}

pip3_chk() {
    if command -v pip3 > /dev/null; then
        PIP3_BIN="pip3"
    elif command -v pip-3.3 > /dev/null; then
        PIP3_BIN="pip-3.3"
    elif command -v pip-3.2 > /dev/null; then
        PIP3_BIN="pip-3.2"
    else
        PIP3_BIN="pip"
    fi
}

pip3_pkg_req() {
    PIP3_CHK=$($PIP3_BIN search $1 | grep INSTALLED)

    if [ "" == "$PIP3_CHK" ]; then
        true
    else
        false
    fi
}

: <<'MAINSTART'

Perform all variables declarations as well as function definition
above this section for clarity, thanks!

MAINSTART

# intro message

if [ $debugmode != "no" ]; then
    if [ $debuguser != "none" ]; then
        gitusername="$debuguser"
    fi
    if [ $debugpoint != "none" ]; then
        gitrepobranch="$debugpoint"
    fi
    newline
    warning "DEBUG MODE ENABLED"
    echo "git user $gitusername and $gitrepobranch branch/tag will be used"
    newline
else
    newline
    echo "This script will install everything needed to use"
    echo "the $topdir $productname"
    newline
    warning "--- Warning ---"
    newline
    echo "Always be careful when running scripts and commands"
    echo "copied from the internet. Ensure they are from a"
    echo "trusted source."
    newline
    echo "If you want to see what this script does before"
    echo "running it, you should run:"
    echo "\curl -sS https://get.pimoroni.com/$scriptname"
    newline
fi

# checks and init

arch_check
os_check
home_dir
check_network

if [ $debugmode != "no" ]; then
    echo "USER_HOME is $USER_HOME"
    newline
fi

if [ "$FORCE" != '-y' ]; then
    space_chk
fi

if ! $IS_ARMHF; then
    warning "This hardware is not supported, sorry!"
    newline
    exit 1
fi

if ! $IS_RASPBIAN && [ $raspbianonly == "yes" ]; then
    warning "Your operating system is not supported, sorry!"
    newline
    exit 1
elif ! $IS_SUPPORTED; then
    warning "Your operating system is not supported, sorry!"
    newline
    exit 1
fi

if [ $forcesudo == "yes" ]; then
    sudocheck
fi

if $IS_RASPBIAN && [ $raspbianonly == "yes" ]; then
    raspbian_check
    if [ $wheezysupport == "no" ] && $IS_WHEEZY || $IS_SQUEEZE; then
        newline
        warning "--- Warning ---"
        newline
        echo "The $productname installer"
        echo "does not work on this version of Raspbian."
        echo "Check https://github.com/$gitusername/$gitreponame"
        echo "for additional information and support"
        newline
        exit 1
    fi
fi

if $IS_ARMv8 && [ $armv8 == "no" ]; then
    warning "Sorry, your CPU is not supported by this installer"
    newline
    exit 1
elif $IS_ARMv7 && [ $armv7 == "no" ]; then
    warning "Sorry, your CPU is not supported by this installer"
    newline
    exit 1
elif $IS_ARMv6 && [ $armv6 == "no" ]; then
    warning "Sorry, your CPU is not supported by this installer"
    newline
    exit 1
fi

if [ $forcesudo == "yes" ]; then
    sudocheck
fi

if [ $pipoverride == "yes" ]; then
    debpackage="na"
else
    piplibname="na"
fi

if [ $i2creq == "yes" ]; then
    echo "Note: $productname requires I2C communication"
    newline
fi
if [ $spireq == "yes" ]; then
    echo "Note: $productname requires SPI communication"
    newline
fi
if [ $uartreq == "yes" ]; then
    echo "Note: $productname requires UART communication"
    warning "The serial console will be disabled if you proceed!"
    newline
fi

if confirm "Do you wish to continue?"; then

# hardware setup

    newline
    echo "Checking hardware requirements..."

    if [ $uartreq == "yes" ]; then
        newline
        echo "The serial console must be disabled for $productname to work"
        \curl -sS https://get.pimoroni.com/uarton | sudo bash -s - "-y"
        ASK_TO_REBOOT=true
        newline
    fi

    if [ $spireq == "yes" ]; then
        newline
        if ls /dev/spi* &> /dev/null; then
            success "SPI already enabled"
        else
            echo "SPI must be enabled for $productname to work"
            \curl -sS https://get.pimoroni.com/spi | sudo bash -s - "-y"
            ASK_TO_REBOOT=true
        fi
    fi

    if [ $i2creq == "yes" ]; then
        newline
        if ls /dev/i2c* &> /dev/null; then
            success "I2C already enabled"
        else
            echo "I2C must be enabled for $productname to work"
            \curl -sS https://get.pimoroni.com/i2c | sudo bash -s - "-y"
            ASK_TO_REBOOT=true
        fi
        newline
        echo "Checking packages required by interface..."
        newline
        if apt_pkg_req "python-smbus" || apt_pkg_req "python3-smbus"; then
            sysupdate
            if apt_pkg_req "python-smbus"; then
                if ! apt_pkg_install "python-smbus"  && python3 --version | grep "2.7" > /dev/null; then            
                    DEBDIR=`mktemp -d /tmp/pimoroni.XXXXXX`
                    cd $DEBDIR
                    wget $RASPOOL/main/i/i2c-tools/python-smbus_3.1.1+svn-2_armhf.deb &> /dev/null
                    sudo dpkg -i $DEBDIR/python-smbus_3.1.1+svn-2_armhf.deb
                fi
            fi
            if apt_pkg_req "python3-smbus"; then
                if ! apt_pkg_install "python3-smbus" && python3 --version | grep "3.4" > /dev/null; then            
                    DEBDIR=`mktemp -d /tmp/pimoroni.XXXXXX`
                    cd $DEBDIR
                    wget $RASPOOL/main/i/i2c-tools/python3-smbus_3.1.1+svn-2_armhf.deb &> /dev/null
                    sudo dpkg -i $DEBDIR/python3-smbus_3.1.1+svn-2_armhf.deb
                fi
            fi
        fi
    fi
    newline

# minimum install routine

    if [ $gitrepoclone != "yes" ]; then

        echo "$productname comes with examples and documentation that you may wish to install."
        echo "Performing a full install with ensure those resources are installed,"
        echo "along with all required dependencies. It may however take a while!"
        newline

        if ! confirm "Do you wish to perform a full install?"; then
            MIN_INSTALL=true
        fi
    fi

    if ! $MIN_INSTALL;then
        if [ $localdir != "na" ]; then
            installdir="$USER_HOME/$topdir/$localdir"
        else
            installdir="$USER_HOME/$topdir"
        fi
        [ -d $installdir ] || mkdir -p $installdir
    fi

    if [ $debugmode != "no" ]; then
        echo "INSTALLDIR is $installdir"
    fi

# apt repo install

    echo "Checking install requirements..."

    if command -v apt-get > /dev/null; then
        newline
        echo "Updating package indexes..."
        sysupdate
        newline
        echo "Checking for dependencies..."

        if $REMOVE_PKG; then
            for pkgrm in ${pkgaptremove[@]}; do
                warning "Installed package conflicts with requirements"
                sudo apt-get remove "$pkgrm"
            done
        fi

        pkgdeplist=( "${pkgdeplist1[@]}" "${pkgdeplist2[@]}" "${pkgdeplist3[@]}" )

        for pkgdep in ${pkgdeplist[@]}; do
            if apt_pkg_req "$pkgdep"; then
                apt_pkg_install "$pkgdep"
            fi
        done
        if raspbian_old; then
            for pkgdep in ${pkgdeplist7[@]}; do
                if apt_pkg_req "$pkgdep"; then
                    apt_pkg_install "$pkgdep"
                fi
            done
        else
            for pkgdep in ${pkgdeplist8[@]}; do
                if apt_pkg_req "$pkgdep"; then
                    apt_pkg_install "$pkgdep"
                fi
            done
        fi
        if [ $debpackage != "na" ]; then
            apt_pkg_install "$debpackage"
        fi
    fi

# pypi repo install

    if [ $pip2support == "yes" ] && [ -n $(which python2) ]; then
        pip2_chk
        for pipdep in ${pipdeplist[@]}; do
            if pip2_pkg_req "$pipdep"; then
                sudo -H $PIP2_BIN install "$pipdep"
            fi
        done
        if [ $piplibname != "na" ] && [ $debpackage == "na" ]; then
            newline
            echo "Installing $productname library for Python 2..."
            newline
            if ! sudo -H $PIP2_BIN install $piplibname -U; then
                warning "Python 2 library install failed!"
                echo "If problems persist, visit forums.pimoroni.com for support"
                exit 1
            fi
        fi
    fi

    if [ $pip3support == "yes" ] && [ -n $(which python3) ]; then
        pip3_chk
        for pipdep in ${pipdeplist[@]}; do
            if pip3_pkg_req "$pipdep"; then
                sudo -H $PIP3_BIN install "$pipdep"
            fi
        done
        if [ $piplibname != "na" ] && [ $debpackage == "na" ]; then
            newline
            echo "Installing $productname library for Python 3..."
            newline
                if ! sudo -H $PIP3_BIN install $piplibname -U; then
                    warning "Python 3 library install failed!"
                echo "If problems persist, visit forums.pimoroni.com for support"
                exit 1
            fi
        fi
    fi

# git repo install

    if [ $gitrepoclone == "yes" ]; then
        if ! command -v git > /dev/null; then
            apt_pkg_install git
        fi
        if [ $gitclonedir == "source" ]; then
            gitclonedir=$gitreponame
        fi
        if [ $repoclean == "yes" ]; then
            rm -Rf $installdir/$gitclonedir
        fi
        if [ -d $installdir/$gitclonedir ]; then
            newline
            echo "Github repo already present. Updating..."
            cd $installdir/$gitclonedir
            git pull
        else
            newline
            echo "Cloning Github repo locally..."
            cd $installdir
            if [ $debugmode != "no" ]; then
                echo "git user name is $gitusername"
                echo "git repo name is $gitreponame"
            fi
            if [ $gitrepobranch != "master" ]; then
                git clone https://github.com/$gitusername/$gitreponame $gitclonedir -b $gitrepobranch
            else
                git clone --depth=1 https://github.com/$gitusername/$gitreponame $gitclonedir
            fi
        fi
    fi

    if [ $repoinstall == "yes" ]; then
        newline
        echo "Installing library..."
        newline
        cd $installdir/$gitreponame/$libdir
        if [ $pip2support == "yes" ] && [ -n $(which python2) ]; then
            sudo python2 ./setup.py install
        fi
        if [ $pip3support == "yes" ] && [ -n $(which python3) ]; then
            sudo python3 ./setup.py install
        fi
        newline
    fi

# additional install

    if ! $MIN_INSTALL; then
        newline
        echo "Checking for additional software..."

        moredepreq=false

        for moredep in ${moreaptdep[@]}; do
            if apt_pkg_req "$moredep"; then
                moredepreq=true
            fi
        done
        if $moredepreq; then
            newline
            for moredep in ${moreaptdep[@]}; do
                if apt_pkg_req "$moredep"; then
                    apt_pkg_install "$moredep"
                fi
            done
            moredepreq=false
        fi

        for moredep in ${morepipdep[@]}; do
            if pip2_pkg_req "$moredep"; then
                moredepreq=true
            fi
            if pip3_pkg_req "$moredep"; then
                moredepreq=true
            fi
        done
        if $moredepreq; then
            newline
            for moredep in ${morepipdep[@]}; do
                if pip2_pkg_req "$moredep"; then
                    sudo -H $PIP2_BIN install "$moredep"
                fi
                if pip3_pkg_req "$moredep"; then
                    sudo -H $PIP3_BIN install "$moredep"
                fi
            done
            moredepreq=false
        fi
    fi

# resources install

    if ! $MIN_INSTALL && [ -n "$copydir" ]; then
        if ! command -v git > /dev/null; then
            apt_pkg_install git
        fi
        newline
        echo "Downloading examples and documentation..."
        newline
        TMPDIR=`mktemp -d /tmp/pimoroni.XXXXXX`
        cd $TMPDIR
        git clone --depth=1 https://github.com/$gitusername/$gitreponame
        cd $installdir
        for repodir in ${copydir[@]}; do
            if [ -d $repodir ] && [ $repodir != "documentation" ]; then
                newline
                if [ -d $installdir/$repodir-backup ]; then
                    rm -R $installdir/$repodir-old &> /dev/null
                    mv $installdir/$repodir-backup $installdir/$repodir-old &> /dev/null
                fi
                mv $installdir/$repodir $installdir/$repodir-backup &> /dev/null
                if [ $gitrepotop != "root" ]; then
                    cp -R $TMPDIR/$gitreponame/$gitrepotop/$repodir $installdir/$repodir &> /dev/null
                else
                    cp -R $TMPDIR/$gitreponame/$repodir $installdir/$repodir &> /dev/null
                fi
                warning "The $repodir directory already exists on your system!"
                echo "We've backed it up to $repodir-backup, just in case you've changed anything!"
            else
                rm -R $installdir/$repodir &> /dev/null
                if [ $gitrepotop != "root" ]; then
                    cp -R $TMPDIR/$gitreponame/$gitrepotop/$repodir $installdir/$repodir &> /dev/null
                else
                    cp -R $TMPDIR/$gitreponame/$repodir $installdir/$repodir &> /dev/null
                fi
            fi
        done
        newline
        success "Resources for your $productname were copied to"
        echo  "$installdir/"
        rm -rf $TMPDIR
    fi

# script custom routines

    if [ $customcmd == "no" ]; then
        if [ -n "$pkgaptremove" ]; then
            newline
            echo "Finalising Install..."
            newline
            sysclean
            newline
        fi
        newline
        success "All done!"
        newline
        echo "Enjoy your $productname!"
        newline
    else # custom block starts here
        newline
        echo "Finalising Install..."
        force_hdmi
        newline
    fi

    if $FAILED_PKG; then
        warning "Some packages could not be installed, review the output for details!"
        newline
    fi

    if [ $promptreboot == "yes" ] || $ASK_TO_REBOOT; then
        sysreboot
        newline
    fi
else
    newline
    echo "Aborting..."
    newline
fi

exit 0
