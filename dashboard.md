# Johdanto

Kokonaan idea lähti toisen henkilön tekemästä projektista, mutta ongelma oli se, että tällä hetkellä ei ole Rasperry PI:tä ja projektiin liittyviä oheistuotteita, [Now Playing: My Raspberry Pi Weekend Project](https://chorus.fm/features/articles/now-playing-my-raspberry-pi-weekend-project/) . Projektissa on luotu vagrant ympäristö, debian/bookworm64 ja salt-master ja 2 salt-minion konetta, flask, python.

Kyseisen projektin idea on näyttää flaskin kautta nettisivu missä näkyy mitä tällä hetkellä kuuntelen spotifyssa, ja myös yrittää saada salt minion lisäämään tämänhetkisen sään samalle, nettisivustolle.




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

- Ihan ensimmäiseksi vagrantin ssh yhteydellä kun olin tmaster koneella kokeilin `ping goole.com` sen vuoksi, että kyseinen ottaa ulospäin yhteyden. Tämän jälkeen `sudo apt update` ja uusi kansio `mkdir spotify-flask-app`

Kansio oli valmis oli aika alkaa rakentaa, 
- `sudo apt install python3.11-venv`
- `python3.11 -m venv venv` [Create a virtualenv and activate it:](https://github.com/pallets/flask/tree/main/examples/tutorial#:~:text=Create%20a%20virtualenv,venv/bin/activate)
- `source venv/bin/activate`
- `pip install flask spotipy python-dotenv` asennetenaan tarvittavat kirjastot
- `nano .env`
  
![image](https://github.com/user-attachments/assets/a1e8244e-083c-4df7-a9b6-9f8d23e9f033)
- tiedostoon lisätty aikaisemmin saatu client id sekä client secret, sekä spotify kohdassa oleva Redirect URIs on myös saatu paikalleen,

Alustus oli nyt tehty, tämän jälkeen aloin tekemään flaskin ja spotifyn yhteyttä. Ohjeistus tähän kohtaan saatu [Display currently playing Song with Spotipy API in Flask.(Soni Kirtan)](https://kirtansoni.medium.com/display-currently-playing-song-with-spotipy-api-in-flask-f0139e63e8bc)

- `nano app.py`
  
 ```yaml
from flask import Flask, redirect, request, session, url_for, jsonify
import spotipy
from spotipy.oauth2 import SpotifyOAuth
from dotenv import load_dotenv
import os

load_dotenv()

app = Flask(__name__)
app.secret_key = os.urandom(24)

SCOPE = 'user-read-currently-playing'

sp_oauth = SpotifyOAuth(
    scope=SCOPE,
    client_id=os.getenv("SPOTIPY_CLIENT_ID"),
    client_secret=os.getenv("SPOTIPY_CLIENT_SECRET"),
    redirect_uri=os.getenv("SPOTIPY_REDIRECT_URI"),
    show_dialog=True,
    cache_path=".cache"  # tallentaa tokenin tiedostoon
)

@app.route('/')
def login():
    auth_url = sp_oauth.get_authorize_url()
    return redirect(auth_url)

@app.route('/callback')
def callback():
    code = request.args.get('code')
    token_info = sp_oauth.get_access_token(code)
    session['token_info'] = token_info
    return redirect('/currently-playing')

@app.route('/currently-playing')
def currently_playing():
    token_info = session.get('token_info', None)
    if not token_info:
        return redirect('/')
    
    sp = spotipy.Spotify(auth=token_info['access_token'])
    current = sp.current_playback()
    if current and current['is_playing']:
        data = {
            'track': current['item']['name'],
            'artist': current['item']['artists'][0]['name'],
            'album': current['item']['album']['name'],
            'image': current['item']['album']['images'][0]['url']
        }
        return jsonify(data)
    return jsonify({'message': 'Nothing playing'})

if __name__ == '__main__':
    app.run(debug=True)
 ```
Tämän jälkeen päätin tehdä etusivun flaskille,

- `mkdir templates`
- `nano templates/index.html`

```yaml
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Spotify Currently Playing</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            text-align: center;
            margin-top: 60px;
        }
        #cover {
            width: 300px;
            border-radius: 10px;
        }
    </style>
</head>
<body>
    <h1>🎵 Nyt soi Spotifyssa</h1>
    <div id="track-info">
        <img id="cover" src="" alt="Album Cover">
        <h2 id="song"></h2>
        <p id="artist"></p>
    </div>

    <script>
        async function fetchTrack() {
            try {
                const response = await fetch('/currently-playing');
                const data = await response.json();

                if (data.is_playing) {
                    document.getElementById('cover').src = data.album_cover;
                    document.getElementById('song').textContent = data.track;
                    document.getElementById('artist').textContent = data.artist;
                } else {
                    document.getElementById('song').textContent = "Ei soittoa juuri nyt 🎧";
                    document.getElementById('artist').textContent = "";
                    document.getElementById('cover').src = "";
                }
            } catch (error) {
                console.error('Virhe noudettaessa kappaletta:', error);
            }
        }

        setInterval(fetchTrack, 5000);
        fetchTrack();
    </script>
</body>
</html>
```

Seuraavaksi oli aika tehdä muutoksia vagrant fileen, 
```yaml
config.vm.define "tmaster", primary: true do |tmaster|
    tmaster.vm.provision :shell, inline: $master
    tmaster.vm.network "private_network", ip: "192.168.12.3"
    tmaster.vm.hostname = "tmaster"

    tmaster.vm.network "forwarded_port", guest: 5000, host: 5000  ◀️ #Lisätty portti
```
Tämän jälkeen piti tottakai käynnistää kone uudeelleen, jotta uusi asetus astuu voimaan,
- `exit`
- `vagrant reload tmaster`
- `vagrant ssh tmaster`
- `cd ~/spotify-flask-app`
- `source venv/bin/activate`

Ensimmäinen ajo näytti tältä ja juuri se mitä haluttiin 💀💀💀💀💀💀💀
![image](https://github.com/user-attachments/assets/2139571e-60f2-43fe-9b3a-b092bbd35657)


- `sudoedit app.py`
Tässä vaiheessa laitoin koodin pätkän chat.gpt:lle ja gpt antoi vaustaukseksi
![image](https://github.com/user-attachments/assets/97c7e682-cee7-4d4c-9c78-07c88a7797c2)


Koodia muokattu

- Toinen ajo näytti tältä, mutta kumminkin jotain oli vielä pielessä, sillä kyseinen ei näyttänyt siltä miltä halusin.
![image](https://github.com/user-attachments/assets/06cd4b2b-e746-48dc-805c-1ff3be0153eb)

Takaisin muokkaamaan koodia, tämän jälkeen koodit menivät aivan uusiksi tokenit yms olivat niin jumissa, että sisään ei päässyt molemmat koodit lyöty chat.gpt ja muokkaukset tehty.

- app.py tiedosto
```yaml
from flask import Flask, redirect, request, session, jsonify, render_template
import spotipy
from spotipy.oauth2 import SpotifyOAuth
from dotenv import load_dotenv
import os


load_dotenv()

app = Flask(__name__)
app.secret_key = os.urandom(24)


SCOPE = 'user-read-currently-playing user-read-playback-state'

sp_oauth = SpotifyOAuth(
    scope=SCOPE,
    client_id=os.getenv("SPOTIPY_CLIENT_ID"),
    client_secret=os.getenv("SPOTIPY_CLIENT_SECRET"),
    redirect_uri=os.getenv("SPOTIPY_REDIRECT_URI"),
    show_dialog=True,
    cache_path=".cache"
)

@app.route('/')
def login():
    auth_url = sp_oauth.get_authorize_url()
    return redirect(auth_url)

@app.route('/callback')
def callback():
    code = request.args.get('code')
    token_info = sp_oauth.get_access_token(code)
    session['token_info'] = token_info
    return redirect('/player')

@app.route('/player')
def player():
    return render_template('index.html')

@app.route('/currently-playing')
def currently_playing():
    token_info = session.get('token_info', None)
    if not token_info:
        return redirect('/')

    if sp_oauth.is_token_expired(token_info):
                token_info = sp_oauth.refresh_access_token(token_info['refresh_token'])
        session['token_info'] = token_info

    try:
        sp = spotipy.Spotify(auth=token_info['access_token'])
        current = sp.current_playback()

        if current and current.get('is_playing'):
            data = {
                'track': current['item']['name'],
                'artist': current['item']['artists'][0]['name'],
                'album': current['item']['album']['name'],
                'image': current['item']['album']['images'][0]['url'],
                'is_playing': True
            }
            return jsonify(data)

        return jsonify({'message': 'Nothing playing', 'is_playing': False})

    except spotipy.exceptions.SpotifyException as e:
        return f"Spotify API error: {e}", 401

if __name__ == '__main__':
    app.run(debug=True, host="0.0.0.0")
```

- index.html
```yaml
<!DOCTYPE html>
<html lang="fi">
<head>
    <meta charset="UTF-8">
    <title>Spotify Currently Playing</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            text-align: center;
            margin-top: 60px;
        }
        #cover {
            width: 300px;
            border-radius: 10px;
        }
    </style>
</head>
<body>
    <h1>🎵 Nyt soi Spotifyssa</h1>
    <div id="track-info">
        <img id="cover" src="" alt="Album Cover">
        <h2 id="song">Ladataan...</h2>
        <p id="artist"></p>
    </div>

    <script>
        async function fetchTrack() {
            try {
                const response = await fetch('/currently-playing');
                const data = await response.json();

                if (data.is_playing) {
                    document.getElementById('cover').src = data.image;
                    document.getElementById('song').textContent = data.track;
                    document.getElementById('artist').textContent = data.artist;
                } else {
                    document.getElementById('song').textContent = "Ei soittoa juuri nyt 🎧";
                    document.getElementById('artist').textContent = "";
                    document.getElementById('cover').src = "";
                }
            } catch (error) {
                console.error('Virhe noudettaessa kappaletta:', error);
            }
        }

        setInterval(fetchTrack, 5000);
        fetchTrack();
 </script>
</body>
</html>
```

Kokeilin uusiksi komentoa
- `flask run --host=0.0.0.0`
- ja menin osoitteeseen `http://localhost:5000/`

  ![image](https://github.com/user-attachments/assets/7ee2fd01-1ad1-47b5-9979-0008db84351e)
  - Hyväksynnän jälkeen sain viimeinkin näkyviin sellaisen mitä odotinkin
    
  ![image](https://github.com/user-attachments/assets/ef56debe-090d-4e04-8f8b-086f47c99313)

Kyseinen idea on vain näyttää, että mikä kappale soi spotifyssa ja albumin kuva, biisi sekä artisti.
- Web player spotify - Toimii
- Pelkkä sovellus - Toimii
Yllä olevat kiinni, tehtävien hallinnasta spotify kokonaan pois päältä
- Puhelin Spotify sovellus Wifi yhteydellä - Toimii
- Puhelin Spotify sovellus pelkällä mobiili yhteydellä - Toimii 


## Weather API & Salt Minion

Ensimmäiseksi piti saada avain [OpenWeather](https://openweathermap.org/api) 
- [Open Weather nettisivu](https://openweathermap.org/api)
- Tilin tekeminen
- Oma tili -> My Api Keys
- Tein uuden avaimen kokonaan
- ![image](https://github.com/user-attachments/assets/a724b9c0-fb14-4526-a927-b7b3d85cbd48)

  
