# IHRIS Installation Using Ansible

Update package list with below command:

```bash
sudo apt update
```

Install git:

```bash
sudo apt install git
```

Install ansible:

```bash
sudo apt install ansible
```

Open file `/etc/ansible/hosts` with your favorite editor and add `127.0.0.1` at the end of the file. If the folder ansible is missing, then create the folder ansible with below command:

```bash
sudo mkdir /etc/ansible
```

Then open the file `/etc/ansible/hosts` and continue with above editing.

Install ansible modules that we will be using:

```bash
sudo ansible-galaxy collection install ansible.utils
```

```sh
sudo ansible-galaxy collection install community.postgresql
```

The installer is inside iHRIS repo, you may partially clone iHRIS repo to get the installer only (recommended) or you may clone the entire iHRIS repo

To partially clone the installer only, run below commands
```bash
git clone -n --depth=1 --filter=tree:0 https://github.com/iHRIS/iHRIS.git
cd iHRIS/
git sparse-checkout set --no-cone packaging
git checkout
```

<b>OR</b> Run Below Command To Clone The Entire iHRIS GitHub repository:

```bash
git clone https://github.com/iHRIS/iHRIS.git
```

Now run the iHRIS ansible installer:

```bash
cd iHRIS/packaging/ansible/ubuntu
```

```bash
sudo bash run.sh
```

After the above command is done running, iHRIS will be installed under the directory `/var/lib/iHRIS`. Use the below commands to stop, start, and restart it:

```bash
sudo service ihris stop
```

```bash
sudo service ihris start
```

```bash
sudo service ihris restart
```

To access iHRIS, open your browser and type the address `localhost:3000` to access iHRIS. The default username is `admin@ihris.org` and the password is `ihris`.
