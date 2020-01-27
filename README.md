## OrderlyWeb / orderly setup for ncov-ic

## Set up a server

Currently using Ubuntu 18.04.  During the installation there is an option along the lines of "allocate 4G to this drive" which it is important to change to "allocate max to this drive".

### Install docker

Install docker following approach [in montagu-machine](https://github.com/vimc/montagu-machine/blob/master/provision/setup-docker)

```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository \
         "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
sudo apt-get update && sudo apt-get install -y docker-ce
sudo usermod -aG docker rich
```

### Add a server user

Create a `server` user - best done with `adduser` (rather than `useradd`)

```
sudo apt-get install -y pwgen
SERVER_PASSWORD=$(pwgen 30 1)
vault write /secret/ncov/server/password value=$SERVER_PASSWORD
sudo adduser --quiet --disabled-password server
echo "server:$SERVER_PASSWORD" | sudo chpasswd
sudo usermod -aG docker server
sudo usermod -aG sudo server
```

### Configure vault on the server

Install vault on the target machine following instructions in [montagu-machine](https://github.com/vimc/montagu-machine/blob/master/provision/setup-vault)

```
vault_version=vault_1.0.3
sudo apt-get install -y unzip
wget https://releases.hashicorp.com/vault/${vault_version}/$vault_zip
unzip $vault_zip
chmod 755 vault
sudo cp vault /usr/bin/vault
rm -f $vault_zip vault
```

And configure a policy for our group:

```
vault policy write ncov_read ncov-vault.hcl
vault write auth/github/map/teams/ncov value=ncov_read
```

## Add the ssl certificates into the vault

```
unzip ncov_cert.zip
cat ncov_dide_ic_ac_uk.crt \
  RootCertificates/QuoVadisOVIntermediateCertificate.crt \
  RootCertificates/QuoVadisOVRootCertificate.crt \
  > ncov.crt
vault write secret/ncov/proxy/ssl_certificate value=@ncov.crt
vault write secret/ncov/proxy/ssl_private_key value=@ncov.key
```

## Create a GitHub organisation

This project used `ncov-ic` for the organisation.  Someone needs to get hold of github to request an academic upgrade to enable private repositories [current documentation on this](https://help.github.com/en/github/teaching-and-learning-with-github-education/applying-for-an-educator-or-researcher-discount#upgrading-your-organization).

## Prepare an orderly repo

Based off, for now, the most recent `ebola-outputs` repo.  Deleted the `.git` directory, plus all content from `src`, searched for terms that referred to the ebola outbreak and replaced them with appropriate ones for 2019-nCoV.  Resulting skeleton repo is [here](https://github.com/ncov-ic/ncov-outputs/tree/866511d).

Add a deploy key to this repository (and to the vault) by running

```
./scripts/add_deploy_key
```

and following the instructions

## Create an oauth app

Two apps are needed - one for "real" and one for testing.

Instructions are on the [orderly-web repo](https://github.com/vimc/orderly-web/blob/master/docs/auth.md#authenticating-with-github)

* GitHub Settings -> Developer Settings ->  New OAuth App (or go to https://github.com/organizations/ncov-ic/settings/applications/new)
* Application name should be the human readable name of the group (e.g., 2019-nCoV, MRC-GIDA, Imperial College)
* Both the homepage URL and the Authorization callback URL should be the name of the orderly instance (https://ncov.dide.ic.ac.uk or https://ncov.dide.ic.ac.uk:1443)
* The next page displays the secrets - set those within the vault

```
vault write secret/ncov/oauth/real \
  id=<id> \
  secret=<secret>
```

and

```
vault write secret/ncov/oauth/testing \
  id=<id> \
  secret=<secret>
```

## Create an orderly container

Starting from the [`ebola-orderly`](https://github.com/imperialebola2018/ebola-orderly) container, search and replace "ebola" strings and replace as appropriate, and push up as `ncov-ic/ncov-orderly`

That needs building on TeamCity.  The easiest way was to go into the [Ebola Orderly](http://teamcity.montagu.dide.ic.ac.uk:8111/admin/editProject.html?projectId=montagu_Orderly_EbolaOrderly) project and select "Copy project" from the "Actions" menu, then edit the VCS root.

Create a new [Docker Hub](https://hub.docker.com/) account.  Hyphens can't be used, so I used `ncovic`.  Add the `vimcrobot` user to the organisation so that images can be pushed from TeamCity.

## Create the orderly web configuration

Starting from the [`ebola-orderly-web`](https://github.com/imperialebola2018/ebola-orderly-web), replace all uses of ebola and update organisations appropriately.

## Create the main server

```
git clone https://github.com/ncov-ic/ncov-orderly
git clone https://github.com/ncov-ic/ncov-orderly-web
sudo apt-get install -y python3-pip
pip3 install --user orderly-web
echo 'export PATH=$PATH:~/.local/bin' >> ~/.profile
cd ncov-orderly-web
orderly-web start config
```

After a minute or two the commands will complete and you can log in at https://ncov.dide.ic.ac.uk

When authorising the OAuth application, be sure to grant it access to the organisation that you are used for auth (as specified in `config/orderly-web.yml`)

## Create the staging server

Starting from the [`staging`](https://github.com/imperialebola2018/staging), replace all uses of ebola and update organisations appropriately.

Follow the instructions in the [`README.md`](https://github.com/ncov-ic/staging/blob/master/README.md) to:

* Install vagrant and virtualbox
* Make sure that the BIOS supports virtualisation
* Bring up the VM

```
git clone https://github.com/ncov-ic/staging staging
cd staging
sudo ./provision/setup-vagrant
sudo ./provision/setup-vault
./scripts/vault-prepare
vagrant up
cp scripts/ssh-testing ~server
```

After a minute or two the commands will complete and you can log in at https://ncov.dide.ic.ac.uk:1443

When authorising the OAuth application, be sure to grant it access to the organisation that you are used for auth (as specified in `config/orderly-web.yml`)
