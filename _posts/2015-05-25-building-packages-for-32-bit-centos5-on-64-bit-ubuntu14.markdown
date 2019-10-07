---
layout: post
title:  "Building packages for 32 bit centos5 on 64 bit ubuntu14"
date:   2015-05-25 22:24:32
---

Sometime you need to create package for architecture, for which you can't have build server. We at Hulu encountered exactly this situation when we had to build unbound dns cache for our xen servers, which are 32 bit centos5 under the hood. We had no centos 5 build server, so we used chroot environment on 64 bit ubuntu. We used following build script:

{% highlight bash %}
#!/bin/bash
set -u -e

INTERNAL_REPO_URL="http://your.private.repo"

release=centos5-i386
build_dir=$(pwd)/build
chroot_dir=$build_dir/$release
sudo rm -rf ${build_dir}/$release
mkdir -p $build_dir
if [ -r ./${release}.tgz ] ; then
    sudo tar xzf ./${release}.tgz -C $build_dir
else
    release_rpm_url=http://mirror.centos.org/centos/5/os/i386/CentOS/centos-release-5-11.el5.centos.i386.rpm
    epel_rpm_url=http://dl.fedoraproject.org/pub/epel/5/i386/epel-release-5-4.noarch.rpm
    release_rpm=/tmp/$(basename $release_rpm_url)
    epel_rpm=/tmp/$(basename $epel_rpm_url)
    cd $build_dir
    wget -q -O $release_rpm $release_rpm_url
    wget -q -O $epel_rpm $epel_rpm_url
    sudo mkdir -p ${chroot_dir}/{var/lib/rpm,etc,etc/rpm,opt,dev}
    echo i686-redhat-linux | sudo tee ${chroot_dir}/etc/rpm/platform
    sudo mkdir -p /etc/rpm
    [ -r /etc/rpm/platform ] && sudo cp /etc/rpm/platform /etc/rpm/platform.real
    echo i686-redhat-linux | sudo tee /etc/rpm/platform
    sudo cp /etc/resolv.conf ${chroot_dir}/etc
    sudo rpm --root ${chroot_dir} --initdb
    sudo rpm -ivh --force-debian --nodeps --root ${chroot_dir} ${release_rpm}
    sudo rpm -ivh --force-debian --nodeps --root ${chroot_dir} ${epel_rpm}
    sudo yum -y --nogpgcheck --installroot=${chroot_dir} install bash yum python-hashlib
    sudo rm /etc/rpm/platform
    [ -r /etc/rpm/platform.real ] && sudo mv /etc/rpm/platform.real /etc/rpm/platform
    cp ${release_rpm} ${chroot_dir}/tmp/
    echo "mknod /dev/urandom c 1 9 && rpm --nodeps -i ${release_rpm}"|sudo chroot ${chroot_dir} bash -s
    cat <<EOF|sudo tee ${chroot_dir}/etc/yum.repos.d/private.repo
[private]
name=private
baseurl=${INTERNAL_REPO_URL}/rpm/centos5/stable/\$basearch
enabled=1
gpgcheck=0
EOF
    echo "yum -y --nogpgcheck update && yum clean all"|sudo chroot ${chroot_dir} bash -s
    sudo tar czf $build_dir/../${release}.tgz ./${release} -C $build_dir
fi
echo "yum install -y make gcc wget python-devel swig openssl-devel expat-devel"|sudo chroot ${chroot_dir} bash -s
sudo find ${chroot_dir}/{usr,etc} -type f |sort >${build_dir}/${release}.list
echo "cd /tmp && wget -q -O - --no-check-certificate https://unbound.net/downloads/unbound-1.4.22.tar.gz|tar xzf - && cd unbound* && ./configure --with-pythonmodule --disable-gost --disable-ecdsa && make && make install"|sudo chroot ${chroot_dir} bash -s
comm -13 ${build_dir}/${release}.list <(sudo find ${chroot_dir}/{usr,etc} -type f |sort)|sed -e "s@${chroot_dir}/@@" >${build_dir}/${release}.fpm.list
fpm --epoch 1 --rpm-user root --rpm-group root -s dir -t rpm -v 1.4.22 -n unbound -C ${chroot_dir} --inputs ${build_dir}/${release}.fpm.list -a i386 -d python -d openssl
{% endhighlight %}
To use this script you may need to install yum and rpm if they are not present on your system. 
{% highlight bash %}
sudo apt-get install -y yum rpm
{% endhighlight %}
To build package we use fpm ruby gem. You can install it using following commands:
{% highlight bash %}
sudo apt-get install -y ruby1.9.3
sudo gem install fpm
{% endhighlight %}

