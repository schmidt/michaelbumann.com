---
date: "2018-10-20T22:04:03+00:00"
draft: false
tags: ["bitcoin", "lightning", "lnd"]
title: "How to install your own bank and payment service - your bitcoin and lightning node"
---
<p>I’ve recently updated and re-installed some of my servers and bitcoin and lightning nodes that I am running. It’s amazing how easy it is to run and operate your own bank and payment service. And I encourage everybody to operate your own bitcoin full-node and lightning node.&nbsp;</p><p>Even though there are plenty of resources out there on how to install everything you need on the various systems, here are a few notes on my setup. -&nbsp; maybe it helps somebody. :)&nbsp;</p><p><b>I am running currently:&nbsp;</b></p><p>* Bitcoin core 0.17.0<br>* lnd 0.5.0-beta <br></p><p>My goal is to have my setup as simple and as default as possible. I am using the default packages where possible and I try to be able to update to latest versions quickly. <br>For parts of the setup I am using some custom ansible scripts which I will not cover here (ansible is rather sooner than later a pain anyway and you should not use it)</p><h3>0. Install basic system packages</h3><p>build-essential, git, unattended-upgrades, vim, zsh,...<br></p><p>make sure to secure your system: ufw, fail2ban, lynis, rkhunter</p><h3>1. Install and configure bitcoind <br></h3><p>The <a href="https://bitcoin.org/en/full-node">how-to run a full node</a> on bitcoin.org has all the information you need to install bitcoind. <br></p><p>Basically installing bitcoind from the bitcoin ubuntu packages repository:<br></p><blockquote><p>sudo apt-add-repository ppa:bitcoin/bitcoin <br>sudo apt-get update<br>sudo apt-get install bitcoind</p></blockquote><p>I am running bitcoind as <i>bitcoin</i> user and have all the bitcoind data and config stored in the <i>bitcoin</i>’s home directory.&nbsp; </p><blockquote><p>useradd -m bitcoin<br>mkdir /home/bitcoin/bitcoind_data</p></blockquote><p>Adjust your bitcoind configuration. You can use the <a href="https://jlopp.github.io/bitcoin-core-config-generator/">config file generator</a> by <a href="https://lopp.net/">Jameson Lopp</a>. You also find all the <a href="https://en.bitcoin.it/wiki/Running_Bitcoin">config options in the wiki</a>.<br><br>This is my <a href="https://gist.github.com/bumi/41dba624756a052dd3470e21ee99c7d2">config file - </a>pretty standard, except the bitcoin datadir and the <a href="http://zeromq.org/">zeromq</a> config that we will need later for lnd. <br></p><p>This is the default <a href="https://github.com/bitcoin/bitcoin/blob/master/contrib/init/bitcoind.service">bitcoin systemd service configuration</a>. By default it is loading the <i>bitcoin.conf</i> from <i>/etc/bitcoin/bitcoin.conf. </i>Though I am also storing the config file in the bitcoin dir <i>/home/bitcoin/bitcoind_data </i>which makes it easier for lnd to find it.<br></p><p>Don’t forget to open the bitcoin port; typically 8333 (<i>ufw allow 8333</i>)</p><p>Now you should be able run bitcoind using systemctl&nbsp;</p><blockquote><p>sudo systemctl start bitcoind<br>sudo systemctl status bitcoind<br></p></blockquote><p>When you want to see the logs in <i>journalctl -u bitcoind</i> you have to make sure that you are not running bitcoind as a <i>daemon</i>, change the <a href="https://www.freedesktop.org/software/systemd/man/systemd.service.html#Type=">Type</a> in the of the systemd service from <i>forking</i> to <i>simple</i> and configure the <i>printtoconsole</i> bitcoind option. Otherwise it does print to console and the logs can not be captured. - But you can just look at the debug.log :) <br></p><p>Also note: when you update the bitcoind installation the systemd service and potentially your changes get overwritten.<br></p><h3>2. Install and configure lnd</h3><p>To run lnd you need go 1.10 or better 1.11. I’ve manually <a href="https://golang.org/dl/">downloaded go</a> and extracted it to <i>/</i><i>usr/lib/go-1.11/</i> <br>Make sure to that you have the go binary in your $PATH. I’ve linked the go bin to <i>/usr/local/bin</i>&nbsp;</p><blockquote><p>sudo ln -s /usr/lib/go-1.11/bin/go /usr/local/bin/go</p></blockquote><p>Also make sure that <i>$GOPATH/bin</i> is in your $PATH for all go installed executables.<br>I’ve added a /<i>etc/profile.d/go.sh</i> with:</p><blockquote><p>export GOPATH=~/go<br>export PATH=$PATH:$GOPATH/bin</p></blockquote><p>Once go is working install lnd as described here in the <a href="https://github.com/lightningnetwork/lnd/blob/master/docs/INSTALL.md#installing-lnd">official installation notes</a>. Make sure to run this from the <i>bitcoin</i> user as we have the default $GOPATH&nbsp; <i>~/go</i> .</p><p>Similar to bitcoind I have LND configured to use <i>/home/bitcoin/lnd_data</i> as main lnd directory. This is my <a href="https://gist.github.com/bumi/c871a0bc082dc9bc9c5d1c19001d9a8d#file-lnd-conf">lnd.conf</a>. <br></p><p>If you want to access your LND node remotely (for example from a mobile app) you need to configure it to rpclisten on a <a href="https://gist.github.com/bumi/c871a0bc082dc9bc9c5d1c19001d9a8d#file-lnd-conf-L5">public interface and the port 10009 (default) must be open.<br></a></p><p>LND needs uses zeromq to read data from bitcoind so make sure you configured <a href="https://gist.github.com/bumi/41dba624756a052dd3470e21ee99c7d2#file-bitcoin-conf-L8">bitcoind with the zeromq config</a> mentioned above.&nbsp;</p><p>And try your lnd setup:</p><blockquote><p>lnd --configfile=/home/bitcoin/lnd_data/lnd.conf</p></blockquote><p>It will tell you to create a wallet using lnci <br></p><blockquote><p>lncli --lnddir=/home/bitcoin/lnd_data create</p></blockquote><p>I use systemd to run lnd as a service. <a href="https://gist.github.com/bumi/c871a0bc082dc9bc9c5d1c19001d9a8d#file-lnd-service">This is my service configuration</a> which goes in <i>/lib/systemd/system/lnd.service </i>(don’t forget to systemctl enable it)<i><br></i>But please note that the wallet needs to be unlocked using the lncli on every start of lnd manually.</p><p><b>That should be it! Your bitcoin and lightning node should be up and running! Now it’s time for the fun part :) <br></b></p><h3>3. Setup your client GUI<br></h3><p>So far I have tried the following lighting apps connected with my LND node:</p><ul><li><a href="https://www.lndthinwallet.com/">Union7 iOS wallet - my favorit mobile wallet so far</a><br></li><li><a href="https://www.zap.jackmallers.com/">ZAP iOS </a><br></li><li><a href="https://github.com/LN-Zap/zap-desktop">ZAP Desktop</a> - nice cross-platform Desktop app</li></ul><p>To connect those client to you wallet you will need the tls.cert and the admin.macaroon. These files can be found in the lnd directory:<br></p><p><i>/home/bitcoin/lnd_data/tls.cert</i> and <i>/home/bitcoin/lnd_data/data/chain/bitcoin/mainnet/admin.macaroon</i></p><p>For ZAP iOS there is <a href="https://github.com/LN-Zap/zapconnect">zapconnect</a> that generates a QR code that can be scanned to configure the mobile client. Union7 has a similar feature <a href="https://www.lndthinwallet.com/">described in the FAQ.</a></p><p>Once connected you can use the client or the lncli to get a new bitcoin address and fund your lightning wallet. <br></p><blockquote><p>lncli --lnddir=/home/bitcoin/lnd_data newaddress p2wkh<br></p></blockquote><p>Now it is time to go shopping! Maybe checkout <a href="https://www.bitrefill.com/">bitrefill,</a> Y’alls or buy some pixels on <a href="https://satoshis.place/">satoshis.place</a><br>All those sites have instructions on how to open channels. Also have a look at explorers like <a href="https://1ml.com/node">1ML</a><a href="https://satoshis.place/"><br></a></p><p>If you have trouble, let me know!&nbsp;</p><p>if this did not help, maybe <a href="https://gist.github.com/bretton/0b22a0503a9eba09df86a23f3d625c13">this write up </a>has the right info for you<br></p>