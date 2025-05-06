### Johdanto

Tässä projektissa otetaan käyttöön monitorointipino, joka koostuu Prometheuksesta, Node Exporterista ja Grafanasta. Toteutus tehdään SaltStackin avulla, joka mahdollistaa palveluiden automaattisen asennuksen ja konfiguroinnin hallitusti ja toistettavasti.

Tavoitteena on:
- Asentaa ja konfiguroida Prometheus-palvelin keräämään mittausdataa
- Käynnistää Node Exporter-palvelu koneen resurssien mittaamiseen
- Visualisoida kerätty tieto Grafanan avulla


Aloitus

- `sudo mkdir -p /srv/salt/monitoring/files`
- `sudo nano top.sls`
- Sisältö
```yaml
base:
  '*':
    - monitoring
```

- `cd /srv/salt/monitoring/files/`
- `sudo nano prometheus.yml`
- Sisältö [Devops With Mike. How to Configure Prometheus yaml file | Prometheus Tutorials | Part 4](https://youtu.be/BD4I09F9jxU?t=190)

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node'
    static_configs:
      - targets: ['localhost:9100']
```


`sudo nano init.sls`
-Sisältö [Installation](https://prometheus.io/docs/prometheus/latest/installation/)
```yaml
prometheus-install:
  pkg.installed:
    - name: prometheus

prometheus-config:
  file.managed:
    - name: /etc/prometheus/prometheus.yml
    - source: salt://monitoring/files/prometheus.yml

prometheus-service:
  service.running:
    - name: prometheus
    - enable: True
```

- Ajo `sudo salt '*' state.apply`

- Tulostus
  
![image](https://github.com/user-attachments/assets/e8981a0c-9572-4405-9eae-a5b7839aa09d)





- `sudo nano init.sls`
-Sisältöä lisätty. 
[Installing Prometheus and Node Exporter](https://www.howtoforge.com/how-to-install-prometheus-and-node-exporter-on-debian-12/#:~:text=the%20Prometheus%20server.-,Installing%20Prometheus%20and%20Node%20Exporter,-Prometheus%20is%20an)
[Grafana Labs. Install Grafana on Debian or Ubuntu](https://grafana.com/docs/grafana/latest/setup-grafana/installation/debian/)
```yaml
prometheus-install:
  pkg.installed:
    - name: prometheus

prometheus-config:
  file.managed:
    - name: /etc/prometheus/prometheus.yml
    - source: salt://monitoring/files/prometheus.yml

prometheus-service:
  service.running:
    - name: prometheus
    - enable: True

node-exporter-service:
  service.running:
    - name: prometheus-node-exporter
    - enable: True

install-prerequisites:
  pkg.installed:
    - pkgs:
      - apt-transport-https
      - software-properties-common
      - wget

import-grafana-gpg-key:
  cmd.run:
    - name: |
        mkdir -p /etc/apt/keyrings && \
        wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
    - unless: test -f /etc/apt/keyrings/grafana.gpg

add-grafana-repository:
  file.append:
    - name: /etc/apt/sources.list.d/grafana.list
    - text: "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main"
    - require:
      - cmd: import-grafana-gpg-key

update-apt:
  cmd.run:
    - name: apt-get update
    - require:
      - file: add-grafana-repository

install-grafana:
  pkg.installed:
    - name: grafana

grafana-service:
  service.running:
    - name: grafana-server
    - enable: True
```

Mitä on tähän mennessä tullut?
- ![image](https://github.com/user-attachments/assets/435ea4a0-5e2b-4057-9e4c-e2aa825aa8dd)

init.sls viimeinen ajo 

![image](https://github.com/user-attachments/assets/4551b29e-99de-4cb0-844d-d7378171cbc1)



Palvelut ovat nyt käynnissä aika selvittää, toimivatko ne verkkoselaimessa.

## Node Exporter
- ![image](https://github.com/user-attachments/assets/111a155f-fa91-44c6-9646-0d6c884f3ca5)
- ![image](https://github.com/user-attachments/assets/d684c122-9592-47ef-8195-103844fe1347)

## Prometheus
- ![image](https://github.com/user-attachments/assets/041203a5-f039-447d-a9ea-19c1d5cf1158)
- ![image](https://github.com/user-attachments/assets/ab49a18f-3985-4138-9310-c46ae12847e4)

## Grafana
- ![image](https://github.com/user-attachments/assets/55ff236c-7b89-4f66-a347-a310d4c682f8)
- ![image](https://github.com/user-attachments/assets/481867c2-b373-4ddf-9e84-127062c96dfd)


# Lähteet

[Kurssi: Palvelinten Hallinta. Tero Karvinen](https://terokarvinen.com/palvelinten-hallinta/)

[Devops With Mike. How to Configure Prometheus yaml file | Prometheus Tutorials | Part 4](https://www.youtube.com/watch?v=BD4I09F9jxU&t=190s&ab_channel=DevopsWithMike)

[Prometheus. Installation](https://prometheus.io/docs/prometheus/latest/installation/)

[howtoforge. Installing Prometheus and Node Exporter](https://www.howtoforge.com/how-to-install-prometheus-and-node-exporter-on-debian-12/#:~:text=the%20Prometheus%20server.-,Installing%20Prometheus%20and%20Node%20Exporter,-Prometheus%20is%20an)

[Grafana](https://grafana.com/)

[Prometheus](https://prometheus.io/)

[Node Exporter](https://prometheus.io/docs/guides/node-exporter/)
