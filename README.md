# macOS Remote Desktop (VNC + Cloudflare Tunnel)

Workflow GitHub Actions che espone un runner **macOS** come desktop remoto usando **Screen Sharing nativo (VNC/ARD)** — nessun software di terze parti sul runner, **nessuna accettazione/click richiesto**: ARD è un daemon di sistema e autorizza la sessione con la sola password VNC. L'accesso dall'esterno avviene tramite **Cloudflare Tunnel**, quindi nessuna porta aperta e nessun IP pubblico.

## Setup una tantum (Cloudflare)

1. Crea un **Cloudflare Tunnel** in Zero Trust → Networks → Tunnels (serve un dominio gestito su Cloudflare).
2. Nel tunnel configura una **Public Hostname**:
   - Subdomain/hostname: es. `vnc.tuodominio.com`
   - Type: **TCP**
   - URL: `tcp://localhost:5900`
3. (Consigliato) Crea un'**Access application** per quell'hostname con una policy (es. email OTP) → il tunnel è privato: solo gli utenti autorizzati possono aprire il tunnel.
4. Copia il **Tunnel token** dalla pagina "Install connector".

## Segreti del repository (Settings → Secrets and variables → Actions)

| Segreto | Obbligatorio | Descrizione |
|---|---|---|
| `CLOUDFLARE_TUNNEL_TOKEN` | sì | Token del tunnel (senza: la run usa un *quick tunnel* con URL casuale `trycloudflare.com`, mostrato nei log pubblici) |
| `VNC_PASSWORD` | consigliato | Password VNC (input fallback: `vnc_password` della run — gli input sono pubblici nella pagina della run) |

> Il protocollo VNC usa **al massimo 8 caratteri** della password. Usa una named tunnel + Cloudflare Access per compensare.

## Avvio

Actions → **macOS Remote Desktop (VNC via Cloudflare Tunnel)** → *Run workflow*:
- `duration`: ore di sessione (1–6, default 6)
- `vnc_hostname`: l'hostname del tunnel (solo per stampare il comando client nel summary)

## Connessione dal tuo computer

```bash
# 1. Installa cloudflared e apri il tunnel (login via browser se hai Access con email OTP)
cloudflared access tcp --hostname vnc.tuodominio.com --url localhost:5900
```

Poi apri **Condivisione schermo** (Finder → *Vai* → *Connetti al server…*, o ⌘K) e connettiti a:

```
vnc://localhost:5900
```

Password: la `VNC_PASSWORD` configurata (prime 8 lettere). Funzionano anche client VNC generici (RealVNC, TigerVNC, Windows VNC Viewer) che supportano la modalità legacy.

## Note

- I runner macOS sono fatturati **10×**: 6 ore ≈ 3.590 minuti. Repo pubblici: minuti gratuiti (purtroppo questo schema è anche un segnale di abuso noto a GitHub).
- Il job viene terminato da GitHub a 360 minuti; il keepalive si ferma a 350 per chiudere pulito.
- Ogni run è una VM effimera: niente persiste, e l'hostname del tunnel resta quello configurato nel dashboard.
- Diagnostica: nel run summary trovi ID/istruzioni; i log del tunnel sono in `/tmp/cloudflared.log` e vengono stampati a ogni avvio/rilancio.
