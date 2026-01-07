# DevOps Project 25-26: Moodle & LTI Integratie

Dit project bevat een volledig gecontaineriseerde setup voor **Moodle 4.4**, geïntegreerd met een custom **Python LTI Tool**, **Jenkins** CI/CD pipeline, en ontsloten via een **Traefik** reverse proxy.


## Projectstructuur

De repository is opgedeeld in logische modules per service. Hieronder zie je de bestandsstructuur:

```text
.
├── jenkins/                 # Jenkins controller & configuratie
│   └── docker-compose.yaml
├── ltitool/                 # Broncode voor de Python LTI-applicatie
│   ├── Dockerfile
│   ├── ltitool.py
│   ├── requirements.txt
│   └── README.md
├── moodle/                  # Productie Moodle stack (App + DB + LTI)
│   └── docker-compose.yaml
├── traefik/                 # Reverse proxy & SSL configuratie
│   ├── certs/               # Plaats hier je mkcert certificaten
│   ├── dynamic/             # Dynamische Traefik config (TLS)
│   └── docker-compose.yaml
├── ltitest.yaml             # Standalone test-stack (zonder Traefik/Jenkins)
└── README.md
```

## Architectuur

De structuur is opgezet met Docker Compose en bestaat uit drie logische stacks die communiceren via een extern `proxy` netwerk:

1.  **Proxy Stack:** Traefik v3.4.5 die zorgt voor routing en TLS-terminatie via `mkcert`.
2.  **Moodle Stack:** De Moodle applicatie gekoppeld aan een MariaDB database en de custom LTI-applicatie.
3.  **Jenkins Stack:** Jenkins controller voor CI/CD en image management.


## Vereisten

  * Docker & Docker Compose
  * [mkcert](https://github.com/FiloSottile/mkcert) (voor lokale SSL-certificaten)
  * Git

## Configuratie & Environment Variables

Dit project maakt gebruik van `.env` bestanden in de subdirectory van elke service om gevoelige configuratie en secrets te beheren. Je moet deze bestanden aanmaken voordat je de containers start.

**Globale Variabele (gebruikt in meerdere stacks):**
| Variabele | Beschrijving |
| :--- | :--- |
| `IP_VM` | Het lokale IP-adres van je machine. Dit wordt gebruikt voor de `traefik.me` DNS routing in de labels. |

### 1\. Traefik (`traefik/.env`)

| Variabele | Beschrijving |
| :--- | :--- |
| `TRAEFIK_DASHBOARD_AUTH` | Basic Auth string voor toegang tot het Traefik dashboard (genereer met `htpasswd`). |

### 2\. Moodle & LTI (`moodle/.env`)

| Variabele | Beschrijving |
| :--- | :--- |
| `MARIADB_ROOT_PASSWORD`| Root wachtwoord voor de database. |
| `MARIADB_PASSWORD` | Wachtwoord voor de `bn_moodle` gebruiker. |
| `SESSION_SECRET` | Secret key voor Flask sessie-ondertekening in de LTI-tool. |
| `OAUTH_SECRET` | Gedeelde secret voor LTI OAuth validatie. |

### 3\. Jenkins (`jenkins/.env`)

| Variabele | Beschrijving |
| :--- | :--- |
| `IP_VM` | Vereist voor de configuratie van de Jenkins URL. |

-----

## Installatie & Deployment

### 1\. SSL Certificaten Genereren

Omdat de setup lokaal draait via `traefik.me`, gebruiken we `mkcert`.
Genereer de certificaten en plaats ze in de map `traefik/certs/`.

```bash
cd traefik/certs
mkcert -key-file key.pem -cert-file cert.pem "*.traefik.me"
```

### 2\. Services Starten (Productie)

Start de stacks in de onderstaande volgorde om netwerkconflicten te voorkomen:

1.  **Start Proxy:**

    ```bash
    cd traefik
    docker compose up -d
    ```

2.  **Start Jenkins:**

    ```bash
    cd ../jenkins
    docker compose up -d
    ```

3.  **Start Moodle & LTI Stack:**

    ```bash
    cd ../moodle
    docker compose up -d
    ```

### 3\. Toegang

Ervan uitgaande dat `IP_VM` ingesteld is op bijvoorbeeld `192-168-1-7`, zijn de services bereikbaar op:

  * **Moodle:** `https://moodle-project2526.192-168-1-7.traefik.me`
  * **LTI Tool:** `https://lti-project2526.192-168-1-7.traefik.me`
  * **Jenkins:** `https://jenkins-project2526.192-168-1-7.traefik.me`
  * **Traefik Dashboard:** `https://dashboard-project2526.192-168-1-7.traefik.me`

-----

## Testomgeving (Standalone)

Voor snelle lokale ontwikkeling of debugging zonder de complexiteit van Traefik en Jenkins, is er een standalone configuratie beschikbaar: `ltitest.yaml`.

Start de testomgeving:

```bash
docker compose -f ltitest.yaml up -d
```

  * **Moodle:** `http://localhost:8080` / `https://localhost:8443`
  * **LTI Tool:** `http://localhost:5000`
  * **Database:** `localhost:3306`

*Opmerking: De test-setup gebruikt hardcoded credentials (zie `ltitest.yaml`) en staat een leeg root-wachtwoord toe voor testgemak.*

## Technische Keuzes

  * **Database:** Er is gekozen voor **MariaDB 10.11.9** in plaats van MySQL 8.0. Tests wezen uit dat MySQL 8.0 connectiviteitsproblemen gaf met de Bitnami Moodle installatiescripts.
  * **LTI Tool:** De tool is geschreven in Python (Flask) en gebruikt een `python:3.11-slim` image om de container lichtgewicht te houden.
  * **Versiebeheer:** De repository bevat duidelijke scheiding tussen broncode (`ltitool`), infrastructuur (`docker-compose` files) en configuratie (`certs`, `dynamic` conf).
