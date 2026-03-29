# Ep.X — Instalare WordyMe pe Proxmox LXC

> 🎬 [Vezi episodul pe YouTube](https://youtube.com/watch?v=LINK)
> ✍️ [Articol pe Substack](https://dorulian.substack.com/p/wordy-me)

---

## Ce facem în acest episod

Instalăm WordyMe, o aplicație open-source de luat notițe, în două moduri: rapid pe PC pentru testare, și pe un server Proxmox într-un Linux Container (LXC) dedicat. Pe drum descoperim câteva capcane legate de rețea și le rezolvăm împreună.

🌐 Vrei să testezi fără instalare? Intră direct pe [wordy.me](https://wordy.me)

---

## Contextul arhitecturii: local vs. server

WordyMe (ca multe aplicații de pe GitHub) este configurată implicit pentru `localhost`. Când o mutăm pe un LXC, browserul tău trebuie să comunice cu un aparat la distanță — fără modificări, aplicația eșuează silențios.

**Ce am schimbat pentru a funcționa pe LXC:**

| Ce | De ce |
|---|---|
| Mapare port `3000` în Docker Compose | Backend-ul era izolat; acum e accesibil din rețea |
| `BETTER_AUTH_URL` = IP real | Backend-ul nu știa să valideze sesiunile de logare de la distanță |
| `VITE_BACKEND_URL` = IP real | Interfața web căuta datele pe `localhost` în loc de server |
| `docker compose up --build` | Forțează „coacerea" adreselor IP în codul sursă |

---

## Varianta 1 — Instalare rapidă pe PC

Cea mai simplă metodă de a testa orice aplicație de pe GitHub, fără să îți „murdărești" sistemul.

### Cerințe

**Windows:**
- [Docker Desktop](https://www.docker.com/products/docker-desktop/)
- Git: `winget install --id Git.Git`

**Mac:**
- [Docker Desktop](https://www.docker.com/products/docker-desktop/)
- Git: rulează `git` în terminal — Mac-ul te va întreba automat dacă vrei Xcode Command Line Tools

### Lansare în 60 de secunde

```bash
git clone https://github.com/TeamCoderz/WordyMe.git
cd WordyMe
docker compose up -d
```

Accesează: `http://localhost:5173`

---

## Varianta 2 — Instalare pe Proxmox LXC

### Pasul 1 — Pregătirea LXC

**Configurație recomandată:**
- Template: `ubuntu-24.04-standard`
- Resurse: 1 CPU, 1GB RAM, 10GB Disk
- Rețea: IP static (ex: `10.10.1.71`)

> ⚠️ **Acțiune critică:** Activează `Options → Features → Nesting` la crearea containerului. Fără această opțiune, Docker nu poate gestiona straturile de fișiere în interiorul LXC.

### Pasul 2 — Instalare Docker

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh ./get-docker.sh
```

### Pasul 3 — Descărcare WordyMe

```bash
apt install git -y
git clone https://github.com/TeamCoderz/WordyMe.git
cd WordyMe
```

### Pasul 4 — Configurare `.env`

```bash
cp .env.example .env
openssl rand -hex 32   # copiază rezultatul pentru BETTER_AUTH_SECRET
nano .env
```

Setări obligatorii în `.env` (înlocuiește `10.10.1.71` cu IP-ul tău):

```env
BETTER_AUTH_SECRET=<codul de 64 caractere generat anterior>
CLIENT_URL=http://10.10.1.71:5173
VITE_BACKEND_URL=http://10.10.1.71:3000
```

### Pasul 5 — Personalizare `docker-compose.yml`

Adaugă în secțiunea `backend`:

```yaml
backend:
  ports:
    - "3000:3000"
  environment:
    - CLIENT_URL=http://10.10.1.71:5173
    - BETTER_AUTH_URL=http://10.10.1.71:3000
```

### Pasul 6 — Lansare

```bash
docker compose up -d --build

# Verificare status
docker ps
```

### Rezultat final

| Serviciu | URL |
|---|---|
| Interfață web | `http://10.10.1.71:5173` |
| Backend API | `http://10.10.1.71:3000` |

---

## Dezinstalare completă

```bash
# Oprește containerele și șterge datele (inclusiv notițele)
docker compose down -v

# Șterge codul sursă
cd ..
rm -rf WordyMe

# Curățare generală Docker (opțional)
docker system prune -a
```

> ℹ️ Folosește `docker compose down` fără `-v` dacă vrei să păstrezi notițele.

**Dezinstalare Docker:**
- Windows/Mac: Uninstall Docker Desktop din meniu Apps
- Linux/LXC: `sudo apt-get purge docker-ce docker-ce-cli containerd.io && sudo rm -rf /var/lib/docker`

---

## Probleme întâmpinate & soluții

| Problemă | Cauză | Soluție |
|---|---|---|
| Login loop / nu pot să mă autentific | `BETTER_AUTH_URL` lipsă sau setat pe `localhost` | Setează IP-ul real al LXC în `.env` și `docker-compose.yml` |
| Frontend nu încarcă date | `VITE_BACKEND_URL` setat pe `localhost` | Setează IP-ul real în `.env`, apoi `docker compose up -d --build` |
| Docker nu pornește în LXC | Opțiunea Nesting dezactivată | Activează Nesting din Features în Proxmox |

---

## Despre WordyMe & TeamCoderz

WordyMe este prima piesă dintr-un ERP complet open-source dezvoltat de [TeamCoderz](https://github.com/TeamCoderz). Tot ce construiesc este open-source — control total asupra datelor tale.

---

## Resurse suplimentare

- [Repo oficial WordyMe](https://github.com/TeamCoderz/WordyMe)
- [Documentație Docker](https://docs.docker.com)
- [Proxmox LXC — documentație](https://pve.proxmox.com/wiki/Linux_Container)

---

*Ultima actualizare: 2026-03-29*