# Projekti 1. Dashboard : Spotify "tämän hetkinen kappale" & Sää yhdellä silmäyksellä



Tämän projektin idea sai alkunsa inspiroivasta Raspberry Pi -pohjaisesta “Now Playing” -projektista, mutta koska käytettävissä ei ollut fyysistä Raspberry Pi:tä tai siihen liittyviä oheislaitteita, päätettiin toteutus tehdä virtuaalisesti Vagrant-ympäristössä. Projektissa rakennettiin Debian-pohjainen järjestelmä, jossa on yksi Salt-master ja kaksi Salt-minionia. Tavoitteena oli kehittää Flask-pohjainen web-sovellus, joka näyttää Spotifyssa parhaillaan soivan kappaleen sekä ajankohtaisen sään Helsingin alueelta. Toteutuksessa hyödynnettiin mm. Spotipy-kirjastoa, OpenWeather API:a ja SaltStackin automaatiota palvelujen hallintaan.

![image](https://github.com/user-attachments/assets/0c57895c-f078-4d43-b5f3-47265e6f11d7)





# Projekti 2. Monitorointipinon käyttöönotto SaltStackilla
Johdanto
Tässä projektissa otetaan käyttöön valvontapino, joka koostuu seuraavista komponenteista:

- Prometheus: kerää mittausdataa
- Node Exporter: tarjoaa järjestelmän resurssien mittaustietoa
- Grafana: visualisoi kerätyn datan

Asennus ja konfigurointi toteutetaan SaltStackin avulla, mikä mahdollistaa automatisoidun, toistettavan ja keskitetysti hallitun käyttöönoton.

Projektin tavoite
- Asentaa ja konfiguroida Prometheus-palvelin
- Käynnistää Node Exporter koneen resurssien mittaamiseen
- Visualisoida mittausdata Grafanan avulla

![image](https://github.com/user-attachments/assets/16887269-2445-4c0c-b03c-c8196878412f)
