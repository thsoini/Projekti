# Johdanto

Idea on saada kokonaan toimiva dashboard.


## Vagrant

Aloitin kokonaan tekemällä uuden kansion Windowsille, ja käytin komentokehotetta komenollan `mkdir tilastokehys` tein kansion `cd tilastokehys` tämän jälkeen loin vagrantfilen käyttämällä `vagrant init`. 

- Vagrant filen tein tämän näköiseksi
```yaml
# -*- mode: ruby -*- 
# vi: set ft=ruby : 

$minion = <<MINION
sudo apt-get update
sudo apt-get -qy install salt-minion
echo "master: 192.168.12.3">/etc/salt/minion
sudo service salt-minion restart
MINION

$master = <<MASTER
sudo apt-get update
sudo apt-get -qy install salt-master
MASTER

Vagrant.configure("2") do |config|
  config.vm.box = "debian/bookworm64"

  config.vm.define "data-collector-1" do |data_collector_1|
    data_collector_1.vm.provision :shell, inline: $minion
    data_collector_1.vm.network "private_network", ip: "192.168.12.101"
    data_collector_1.vm.hostname = "data-collector-1"
  end

  config.vm.define "data-collector-2" do |data_collector_2|
    data_collector_2.vm.provision :shell, inline: $minion
    data_collector_2.vm.network "private_network", ip: "192.168.12.102"
    data_collector_2.vm.hostname = "data-collector-2"
  end

  config.vm.define "tmaster", primary: true do |tmaster|
    tmaster.vm.provision :shell, inline: $master
    tmaster.vm.network "private_network", ip: "192.168.12.3"
    tmaster.vm.hostname = "tmaster"
  end
end

```

Kun config tiedosto oli valmis oli aika lyödä koneet käyntiin, komentokehote auki, `vagrant up`

- ![image](https://github.com/user-attachments/assets/bef3ce36-905c-4d01-9746-05b4429bf87c)
- Koneet ovat käynnistyneet ilman ongelmia.

## Salt

Kun koneet oli päällä menin yksitellen kullekkin koneelle ja asensin salt paketit. [Salt Project - Install Salt DEBs](https://docs.saltproject.io/salt/install-guide/en/latest/topics/install-by-operating-system/linux-deb.html#:~:text=Install%20Salt%20DEBs,sources%20%7C%20sudo)
- `vagrant ssh [koneen nimi]`
- `mkdir -p /etc/apt/keyrings`
- `sudo apt install curl`
- `curl -fsSL https://packages.broadcom.com/artifactory/api/security/keypair/SaltProjectKey/public | sudo tee /etc/apt/keyrings/salt-archive-keyring.pgp`
- `curl -fsSL https://github.com/saltstack/salt-install-guide/releases/latest/download/salt.sources | sudo tee /etc/apt/sources.list.d/salt.sources`
- `sudo apt update`
  
Kun salt oli onnistuneesti ladattu koneille oli aika ladata tmaster koneelle salt-master ja 2 muulle koneelle salt-minion
Master koneella
- `sudo apt install salt-master`
- `sudo systemctl enable salt-master && sudo systemctl start salt-master`

Minion koneilla
- `sudo apt install salt-minion`
- `sudoedit /etc/salt/minion` Filen sisään masterin ip osoite sekä id: keraaja1 & keraaja2
- `sudo systemctl enable salt-minion && sudo systemctl start salt-minion`

Avainten hyväksyntä
- `vagrant ssh tmaster`
- `sudo salt-key -L` Avaimet tulivat näkyviin
- ![image](https://github.com/user-attachments/assets/438bc7dd-6816-432b-95a8-c7b03c92e056)
- `sudo salt-key -A` Avaimet hyväksytty

Ping testi
- `sudo salt '*' test.ping`
- ![image](https://github.com/user-attachments/assets/66a59374-99a9-47ef-bfea-9faa2af829ec)


## Spotify 
Spotify ohjelma pitää luoda erikseen spotifyn sivustolla [Spotify for Developers](https://developer.spotify.com/dashboard), painoin painiketta Create app 

![image](https://github.com/user-attachments/assets/a04c6e83-3104-4b35-828c-b8a617714c86)

Luomisen jälkeen otin talteen Client ID ja Client Secret,

![image](https://github.com/user-attachments/assets/1f261ab0-6882-45f6-ad70-525a669e58fc)


## Flask

- Ihan ensimmäiseksi vagrantin ssh yhteydellä kun olin tmaster koneella kokeilin `ping goole.com` sen vuoksi, että kyseinen ottaa ulospäin yhteyden. Tämän jälkeen `sudo apt update`

vagrant@tmaster:~$ mkdir spotify-flask-app
vagrant@tmaster:~$ cd spotify-flask-app
vagrant@tmaster:~/spotify-flask-app$


