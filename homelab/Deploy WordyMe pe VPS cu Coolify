Toate comenzile folosite în clip, în ordinea apariției.

---

## 1. Generare cheie SSH (dacă nu ai deja)

```bash
ssh-keygen -t ed25519 -C "wordyme-vps"
cat ~/.ssh/id_ed25519.pub
```

Copiază outputul și adaugă-l în Hetzner la crearea serverului.

---

## 2. Conectare la VPS

```bash
ssh root@<IP-VPS>
```

Înlocuiește `<IP-VPS>` cu IP-ul primit de la Hetzner.

---

## 3. Instalare Coolify

```bash
curl -fsSL https://cdn.coollabs.io/coolify/install.sh | bash
```

Instalează automat: Docker, Docker Compose, Coolify, Traefik.

---

## 4. Generare secret pentru variabilele de mediu

```bash
openssl rand -base64 32
```

Folosit pentru `BETTER_AUTH_SECRET` în Coolify.

---

## 5. Verificare variabile în container (după deploy)

```bash
docker exec $(docker ps | grep backend | awk '{print $1}') env | grep BETTER
```

Confirmă că variabilele au ajuns corect la container.

---

## 6. Verificare propagare DNS

```bash
ping wordyme.dorulian.eu
```

Când răspunde cu IP-ul VPS-ului, DNS-ul s-a propagat și ești gata de deploy.

---

## Variabile de mediu necesare în Coolify

| Variabilă | Valoare |
|---|---|
| `CLIENT_URL` | `https://<domeniu>` |
| `BETTER_AUTH_URL` | `https://<domeniu>` |
| `BETTER_AUTH_SECRET` | output din `openssl rand -base64 32` |
| `VITE_BACKEND_URL` | *(lasă gol)* |

> **Important:** după fiecare variabilă apasă **Update** în Coolify — altfel nu se salvează.
