# Ghid Tehnic: Instalare WordyMe pe Proxmox (LXC + Docker)

Acest document conține pașii exacți parcurși pentru instalarea și configurarea WordyMe întrun container LXC Ubuntu 24.04 pe Proxmox.

---

## 💡 Contextul Arhitecturii: Local vs. Server (LXC)

Este important de înțeles că WordyMe (ca multe alte aplicații de pe GitHub) este configurată implicit pentru a rula pe un **PC personal (localhost)**.

### Problema de bază:

Când o aplicație rulează pe `localhost`, browserul și serverul sunt pe același aparat. Când o mutăm pe un **Server/LXC**, browserul tău (pe laptop) trebuie să comunice cu un aparat la distanță prin rețea. Fără modificări, aplicația va încerca să se caute pe ea însăși pe laptopul tău și va eșua.

### Ce am modificat pentru a funcționa pe LXC:

1. **Maparea Porturilor:** Am deschis manual portul `3000` în Docker Compose. Implicit, backend-ul era izolat; acum este accesibil din rețea.
2. **Identitatea Serverului (`BETTER_AUTH_URL`):** I-am spus backend-ului adresa sa reală de IP. Fără asta, el nu știe să valideze sesiunile de logare venite de la distanță.
3. **Busola Browserului (`VITE_BACKEND_URL`):** Am instruit interfața web (care rulează în browserul tău) să caute datele la IP-ul serverului (`10.10.1.71`), nu pe `localhost`.
4. **Reconstrucția (`--build`):** Am forțat Docker să re-asambleze aplicația pentru a „coace" aceste adrese de IP direct în codul sursă.

---

## Pasul 1: Pregătirea Proxmox LXC

Am creat un container LXC ultra-eficient pentru a găzdui aplicația.

**Configurație:**

- Template: `ubuntu-24.04-standard`
- Resurse: 1 CPU, 1GB RAM, 10GB Disk
- Rețea: IP Static `10.10.1.71`
- **Acțiune Critică:** Am activat `Options -> Features -> Nesting`. _De ce?_ Permite Docker să gestioneze straturile de fișiere în interiorul containerului Proxmox.

> 🎬 **Notiță filmare:** Arată opțiunea Nesting în interfața Proxmox înainte să creezi containerul. Spune explicit: _„Dacă uiți să activezi asta, Docker nu va porni și vei pierde timp să îți dai seama de ce."_ E un moment vizual clar pe care spectatorul îl poate replica imediat.

---

## Pasul 2: Instalare Docker

Am folosit scriptul oficial Docker pentru o instalare rapidă și corectă pe Ubuntu.

**Comenzi:**

```bash
# Descarcare si rulare script oficial
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh ./get-docker.sh
```

_De ce?_ Docker izolează aplicația (Frontend și Backend) de sistemul de operare principal, asigurând portabilitatea.

> 🎬 **Notiță filmare:** Menționează că acesta e scriptul oficial Docker, nu ceva random de pe internet — contează pentru cei care sunt prudenți cu ce rulează ca root. Poți adăuga: _„Îl descărcăm mai întâi și îl rulăm separat, tocmai ca să putem vedea ce face dacă vrem."_

---

## Pasul 3: Descărcare WordyMe

Am adus codul sursă direct de pe GitHub.

**Comenzi:**

```bash
# Instalare git si clonare repo
apt install git -y
git clone https://github.com/TeamCoderz/WordyMe.git
cd WordyMe
```

> 🎬 **Notiță filmare:** Arată repo-ul pe GitHub înainte să clonezi — caută fișierul `Dockerfile` sau `docker-compose.yml` și spune: _„Dacă o aplicație de pe GitHub are acest fișier, înseamnă că tot ce facem astăzi funcționează și pentru ea."_ E momentul în care generalizezi lecția pentru spectator.

---

## Pasul 4: Configurare Variabile de Mediu (.env)

Configurăm aplicația să recunoască adresa serverului Proxmox.

**Comenzi:**

```bash
# Creare fisier din modelul oferit
cp .env.example .env

# Generare Secret de Securitate
openssl rand -hex 32

# Editare manuala (nano .env)
# Setari obligatorii:
BETTER_AUTH_SECRET=[Codul de 64 caractere generat anterior]
CLIENT_URL=http://10.10.1.71:5173
VITE_BACKEND_URL=http://10.10.1.71:3000
```

> 🎬 **Notiță filmare:** Trei momente de explicat verbal:
> 
> - La `cp .env.example .env` — _„Nu editez fișierul original, îl copiez. Așa îl pot reseta oricând dacă greșesc ceva."_
> - La `openssl rand -hex 32` — _„Ăsta e secretul de securitate al aplicației. Trebuie să fie unic — nu folosi ce vezi pe ecranul meu."_
> - La `VITE_BACKEND_URL` — _„Îi spunem interfeței web unde să găsească datele. Dacă lași localhost aici, aplicația funcționează pe server dar browserul tău nu o poate accesa."_

---

## Pasul 5: Personalizare Docker Compose (Rezolvare Login Loop)

Am adaptat planul de lansare (`docker-compose.yml`) pentru a deschide accesul către API.

**Modificări în `docker-compose.yml` (secțiunea backend):**

```yaml
backend:
  ports:
    - "3000:3000"
  environment:
    - CLIENT_URL=http://10.10.1.71:5173
    - BETTER_AUTH_URL=http://10.10.1.71:3000
```

> 🎬 **Notiță filmare:** 
> docker ps
> Spune: _„Implicit, backend-ul e invizibil din rețea. Prin această modificare îi deschidem o ușă. Și prin `BETTER_AUTH_URL` îi spunem cine este el cu adevărat — altfel nu știe să valideze logările venite de pe alt aparat."_ Dacă ai timp, arată ce se întâmplă fără această modificare — login loop-ul e mai convingător decât orice explicație.

---

## Pasul 6: Lansare și Reconstrucție

Am pornit containerele și am forțat aplicarea noilor setări.

**Comandă:**

```bash
# Pornire cu reconstructie imagini
docker compose up -d --build

# Verificare status
docker ps
```

> 🎬 **Notiță filmare:** Explică `--build` în două propoziții: _„Fără acest flag, Docker folosește versiunea veche a aplicației și modificările noastre din `.env` nu sunt aplicate. `--build` îl forțează să reconstruiască totul de la zero cu noile setări."_ Apoi arată `docker ps` și confirmă că ambele containere (frontend și backend) au statusul `Up`.

---

## Rezultat Final

- **Interfață Web:** `http://10.10.1.71:5173`
- **Backend API:** `http://10.10.1.71:3000`
- **Status:** Aplicația este complet funcțională și accesibilă din browserul local.

---

## 🚀 Alternativa: Instalare Rapidă pe PC (Quick Start)

Dacă vrei să testezi aplicația pe propriul laptop (Windows sau Mac), aceasta este **cea mai simplă metodă de a testa aproape orice aplicație de pe GitHub**.

### 1. Cerințe Prealabile (Instalare Git & Docker)

Dacă nu ai deja uneltele instalate:

- **Pe Windows:**
    - **Docker:** Descarcă [Docker Desktop](https://www.docker.com/products/docker-desktop/).
    - **Git:** Deschide terminalul (CMD/PowerShell) și scrie: `winget install --id Git.Git`.
- **Pe Mac:**
    - **Docker:** Descarcă [Docker Desktop](https://www.docker.com/products/docker-desktop/).
    - **Git:** Deschide terminalul și scrie `git`. Dacă nu e instalat, Mac-ul te va întreba automat dacă vrei să instalezi "Xcode Command Line Tools". Spune "Yes".
- **Pe Linux (Ubuntu/Debian):**
    - **Docker:** `curl -fsSL https://get.docker.com -o get-docker.sh && sudo sh ./get-docker.sh`
    - **Git:** `sudo apt install git -y`

### 2. Lansarea în 60 de secunde

După ce ai uneltele, rulează aceste comenzi în folderul unde vrei să stea proiectul:

```bash
git clone https://github.com/TeamCoderz/WordyMe.git
cd WordyMe
docker compose up -d
```

Accesează apoi: `http://localhost:5173`.

---

## 🧹 Cum cureți totul (Dezinstalare completă)

Dacă ai terminat testarea și vrei să ștergi totul de pe PC/Server:

1. **Oprește și șterge containerele & datele:** În folderul proiectului (`WordyMe`), scrie:
    
    ```bash
    docker compose down -v
    ```
    
    _Notă: Parametrul `-v` (volumes) șterge și baza de date (notițele). Dacă vrei să le păstrezi, folosește doar `docker compose down`._
    
2. **Șterge folderul cu codul sursă:**
    
    ```bash
    cd ..
    rm -rf WordyMe
    ```
    
3. **Curățare generală Docker (opțional):** Pentru a șterge imaginile descărcate și cache-ul rămas:
    
    ```bash
    docker system prune -a
    ```
    
4. **Dezinstalare Docker (Dacă nu mai ai nevoie de el):**
    
    - **Windows/Mac:** Uninstall "Docker Desktop" din meniul de Apps.
    - **LXC (Linux):** `sudo apt-get purge docker-ce docker-ce-cli containerd.io` urmat de `sudo rm -rf /var/lib/docker`.

---

## ✅ Rezultat Final

Docker este **cel mai bun mod de a experimenta cu software-ul de pe GitHub** fără să îți „murdărești" sistemul principal cu instalări de baze de date sau limbaje de programare. Totul stă în „containere" pe care le poți șterge instantaneu, exact cum am făcut mai sus.

---

## Resurse suplimentare

- [Repo oficial WordyMe](https://github.com/teamcoderz/wordyme)
- [Documentație Docker](https://docs.docker.com/)
- [Proxmox LXC — documentație](https://pve.proxmox.com/wiki/Linux_Container)

---

_Ultima actualizare: 2026-03-29_