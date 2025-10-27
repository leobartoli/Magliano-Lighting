# 💡 Smart Lighting Management - Comune di Magliano in Toscana

Sistema di gestione centralizzata e intelligente dell’illuminazione pubblica per i Beni Storici di **San Bruzio** e delle **Mura Medievali**. Architettura Zero Trust con IA locale per sovranità del dato e conservazione del patrimonio culturale.

-----

## 📋 Indice

- [Architettura del Sistema](#-architettura-del-sistema)
- [Stack Tecnologico](#-stack-tecnologico)
- [Integrazione Casambi](#-integrazione-casambi)
- [Agente IA Locale](#-agente-ia-locale-ollama)
- [Prompt Engineering](#-prompt-engineering)
- [Deployment](#-deployment)
- [Sicurezza](#-sicurezza)

-----

## 🏗️ Architettura del Sistema

Il sistema è progettato per garantire **sovranità del dato** e **analisi locale** delle logiche di illuminazione, evitando l’invio di dati sensibili a servizi cloud esterni.

### Principi Architetturali

1. **Zero Trust Network**: Accesso amministrativo esclusivamente via ZeroTier VPN
1. **Edge Computing**: Elaborazione IA locale tramite Ollama su Docker
1. **API-First**: Integrazione Casambi Cloud solo per attuazione comandi
1. **Data Sovereignty**: Dati diagnostici e storici mantenuti on-premise

### Diagramma Logico

```
┌─────────────────────────────────────────────────────────┐
│                 LIVELLO 1: Beni Storici                 │
├─────────────────────────────────────────────────────────┤
│  San Bruzio              │         Mura Medievali       │
│  • Proiettori RGBW       │         • Apparecchi DALI    │
│  • Faretti DALI          │         • Illuminazione diff.│
│         ↓                │                ↓              │
│    Salvador 1064         │         Salvador 1064         │
│    (DALI Controller)     │         (DALI Controller)     │
└──────────────┬───────────────────────────┬───────────────┘
               │      Router 4G LTE        │
               └───────────┬───────────────┘
                           ↓
               ┌───────────────────────┐
               │   Casambi Cloud API   │
               │   (REST + WebSocket)  │
               └───────────┬───────────┘
                           ↓
┌─────────────────────────────────────────────────────────┐
│          LIVELLO 2: Server Comunale (Docker)            │
├─────────────────────────────────────────────────────────┤
│  ┌──────────────┐    ┌──────────────┐                  │
│  │     n8n      │◄───┤    Ollama    │                  │
│  │ (Workflow)   │    │  (IA Locale) │                  │
│  │              │    │  • Llama 3.1 │                  │
│  │ • REST API   │    │  • Mistral   │                  │
│  │ • WebSocket  │    └──────────────┘                  │
│  │ • Scheduler  │                                       │
│  └──────┬───────┘                                       │
│         │                                                │
│  ┌──────▼─────────────────────────┐                    │
│  │  PostgreSQL / SQLite           │                    │
│  │  (Dati Storici & Diagnostica)  │                    │
│  └────────────────────────────────┘                    │
└─────────────────────────────────────────────────────────┘
               ↕                    ↕
    ┌──────────────────┐  ┌─────────────────┐
    │  ZeroTier VPN    │  │ Cloudflare      │
    │  (Admin Access)  │  │ Tunnel          │
    │                  │  │ (Public Webhook)│
    └──────────────────┘  └─────────────────┘
```

-----

## 🔧 Stack Tecnologico

### Livello Infrastrutturale

|Componente           |Versione|Ruolo                  |Configurazione                                   |
|---------------------|--------|-----------------------|-------------------------------------------------|
|**Docker**           |24.x+   |Container Runtime      |Volume persistenti per DB e modelli Ollama       |
|**n8n**              |1.x     |Orchestrazione Workflow|Modalità `queue`, webhook pubblici via Cloudflare|
|**Ollama**           |0.3.x+  |LLM Runtime            |Modelli: `llama3.1:8b`, `mistral:7b`             |
|**ZeroTier**         |1.12.x  |VPN Layer 2            |Network ID privato, ACL restrittive              |
|**Cloudflare Tunnel**|Latest  |Reverse Proxy          |Autenticazione JWT per webhook pubblici          |

### Livello Illuminazione

|Componente     |Fornitore|Quantità|Specifiche                              |
|---------------|---------|--------|----------------------------------------|
|Proiettori RGBW|Domolight|Rif. B  |DALI-2, CRI >90, IP65                   |
|Faretti DALI   |Domolight|Rif. A/C|3000K, dimming 1-100%                   |
|Salvador 1064  |Casambi  |2 unità |Bridge DALI-Bluetooth, 64 indirizzi DALI|
|Cloud Gateway  |Casambi  |1 unità |Connettività LTE/Ethernet al Cloud      |

**Riferimenti Preventivo**: [Domolight Quote - San Bruzio/Mura]

-----

## 🔌 Integrazione Casambi

### 2.1 Autenticazione e Sessione

**Endpoint**: `POST https://api.casambi.com/v1/networks/userlogin`

```json
{
  "email": "comune@magliano.it",
  "password": "***",
  "type": "client"
}
```

**Response**:

```json
{
  "sessionId": "abc123...",
  "userId": "...",
  "networks": [
    {
      "id": "network_san_bruzio",
      "name": "San Bruzio"
    }
  ]
}
```

**Implementazione n8n**:

- Nodo `HTTP Request` con credenziali in variabili d’ambiente
- Cache `sessionId` in memoria workflow (timeout 24h)
- Refresh automatico su codice `401`

-----

### 2.2 Raccolta Dati Diagnostici (REST)

**Endpoint**: `GET https://api.casambi.com/v1/networks/{networkId}/datapoints`

**Query Parameters**:

```
?sensorType=energy,diagnostics
&from=2025-10-20T00:00:00Z
&to=2025-10-27T23:59:59Z
```

**Output Rilevante per IA**:

```json
{
  "datapoints": [
    {
      "unitId": "0x1A",
      "timestamp": "2025-10-26T23:15:00Z",
      "energyWh": 145.2,
      "status": "online",
      "dimLevel": 0.75,
      "colorTemp": 3000
    },
    {
      "unitId": "0x2B",
      "status": "offline",
      "lastSeen": "2025-10-26T21:30:00Z"
    }
  ]
}
```

**Workflow n8n**:

1. Schedule Node (ogni 15 minuti)
1. HTTP Request → Casambi API
1. JSON Parse & Store → PostgreSQL
1. Trigger condizionale: se `status="offline"` → Ollama Diagnostics Prompt

-----

### 2.3 Controllo Real-Time (WebSocket)

**Connessione**: `wss://api.casambi.com/v1/ws`

**Handshake**:

```json
{
  "method": "open",
  "id": 1,
  "session": "abc123...",
  "wire": 83,
  "type": 1
}
```

**Comando Dimming (Inviato da n8n)**:

```json
{
  "method": "controlUnit",
  "id": 2,
  "targetId": "0x1A",
  "level": 0.60,
  "transitionTime": 2.0
}
```

**Comando Scenario RGBW**:

```json
{
  "method": "controlUnit",
  "id": 3,
  "targetId": "0x3C",
  "rgb": [0, 0, 128],
  "white": 255,
  "level": 1.0
}
```

**Eventi in Ingresso** (Listener n8n):

```json
{
  "method": "unitChanged",
  "online": false,
  "unitId": "0x2B",
  "timestamp": "2025-10-27T01:23:45Z"
}
```

**Azione Automatica**:
→ Trigger workflow `diagnostica_predittiva.json`  
→ Input Ollama: cronologia guasti + contesto meteo  
→ Output: probabilità guasto imminente + azioni preventive

-----

## 🧠 Agente IA Locale (Ollama)

### 3.1 Configurazione Docker

**docker-compose.yml**:

```yaml
version: '3.8'

services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama-magliano
    volumes:
      - ./ollama-models:/root/.ollama
    ports:
      - "11434:11434"
    environment:
      - OLLAMA_HOST=0.0.0.0
    restart: unless-stopped
    networks:
      - magliano-net

  n8n:
    image: n8nio/n8n:latest
    container_name: n8n-magliano
    depends_on:
      - ollama
    environment:
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - WEBHOOK_URL=https://lighting.magliano.it
      - OLLAMA_API_URL=http://ollama:11434
    volumes:
      - ./n8n-data:/home/node/.n8n
    ports:
      - "5678:5678"
    networks:
      - magliano-net

networks:
  magliano-net:
    driver: bridge
```

**Modelli Installati**:

```bash
docker exec ollama-magliano ollama pull llama3.1:8b
docker exec ollama-magliano ollama pull mistral:7b-instruct
```

-----

### 3.2 Comunicazione n8n → Ollama

**Nodo HTTP Request in n8n**:

- **URL**: `http://ollama:11434/api/generate`
- **Method**: POST
- **Headers**: `Content-Type: application/json`

**Payload Tipo**:

```json
{
  "model": "llama3.1:8b",
  "prompt": "{{ $json.systemPrompt }}\n\nInput:\n{{ $json.inputData }}",
  "stream": false,
  "options": {
    "temperature": 0.3,
    "top_p": 0.9
  }
}
```

**Response Ollama**:

```json
{
  "model": "llama3.1:8b",
  "response": "{\"recommendedAction\":\"dimming\",\"dimLevel\":0.60,...}",
  "done": true,
  "total_duration": 2340000000
}
```

**Parsing n8n**:

```javascript
// Code Node
const ollamaResponse = JSON.parse($input.item.json.response);
return {
  json: {
    action: ollamaResponse.recommendedAction,
    params: ollamaResponse
  }
};
```

-----

## 📝 Prompt Engineering

### 4.1 Prompt: Dimming Ordinario Notturno

**Contesto**: Illuminazione generale dei Beni Storici nelle ore notturne (22:00 - 06:00) con assenza di traffico pedonale.

**Obiettivo**: Bilanciare risparmio energetico e conservazione dell’impatto visivo del monumento.

```markdown
# RUOLO
Sei un consulente di conservazione dei beni culturali e lighting designer specializzato in illuminazione architettonica storica. Il tuo obiettivo primario è preservare l'integrità visiva e fisica del Bene Storico illuminato, minimizzando lo stress termico e luminoso sulla struttura.

# CONTESTO
Stai gestendo l'illuminazione notturna di:
- **San Bruzio**: Chiesa romanica del XII secolo, facciate in pietra calcarea
- **Mura Medievali**: Struttura fortificata del XIV secolo, pietra arenaria

# VINCOLI TECNICI
- Sistema DALI con dimming continuo (0.01 - 1.00)
- Temperatura colore fissa: 3000K (bianco caldo)
- CRI minimo: 90 (fedeltà cromatica essenziale per valorizzazione architettonica)
- Livello di riferimento testato: 0.85 (piena visibilità notturna, test 15/10/2025)

# VINCOLI NORMATIVI
- Zona UNESCO: riduzione inquinamento luminoso obbligatoria
- Orario 01:00-05:00: riduzione minima 20% richiesta da Soprintendenza
- Mantenimento leggibilità architettonica: livello mai sotto 0.50

# ISTRUZIONI
Analizza l'input JSON e calcola il livello di dimming ottimale (dimLevel: float 0.50-1.00) considerando:
1. **Ora corrente** vs. presenza stimata visitatori
2. **Traffico pedonale** (sensori o stime storiche)
3. **Condizioni meteo** (se disponibili): nebbia/pioggia → +10% luminosità
4. **Priorità conservazione**: evitare cicli on/off frequenti (stress lampade DALI)

# OUTPUT RICHIESTO
Rispondi SOLO con JSON valido, senza markdown:

{
  "recommendedAction": "dimming",
  "dimLevel": <float 0.50-1.00>,
  "colorTemp": 3000,
  "transitionTime": <secondi>,
  "justification": "<max 150 caratteri>",
  "energySaving": "<percentuale stimata>"
}
```

**Input Esempio (da n8n)**:

```json
{
  "context": "dimmingOrdinarioStorico",
  "beneStorico": "San Bruzio",
  "oraCorrente": "02:15",
  "livelloRiferimento": 0.85,
  "trafficoPedonale": "Assente",
  "condMeteo": "Sereno",
  "temperaturaAria": 12,
  "vincoliArchitettonici": "Facciata Sud: riduzione preferita per minimizzare esposizione cumulativa"
}
```

**Output Atteso (Ollama)**:

```json
{
  "recommendedAction": "dimming",
  "dimLevel": 0.55,
  "colorTemp": 3000,
  "transitionTime": 120,
  "justification": "Riduzione a 0.55 per traffico nullo e ora notturna, mantenendo visibilità architettonica. Transizione lenta (2min) per stress termico minimo.",
  "energySaving": "35%"
}
```

-----

### 4.2 Prompt: Scenari Eventi Istituzionali

**Contesto**: Illuminazione dinamica RGBW per ricorrenze civili e religiose.

**Obiettivo**: Tradurre il tema emotivo dell’evento in parametri tecnici RGBW rispettosi della dignità storica.

```markdown
# RUOLO
Sei un direttore artistico specializzato in illuminazione scenografica per monumenti storici. Devi tradurre temi emozionali in configurazioni tecniche RGBW mantenendo sempre la solennità e il rispetto del contesto storico-culturale.

# CONTESTO TECNICO
**Apparecchiature RGBW** (Preventivo Domolight Rif. B):
- Proiettori 4 canali: Red, Green, Blue, White (DALI-2)
- Mixing additivo: RGB per colore, W per luminosità base
- CRI >95 in modalità White puro
- Installazione: facciate principali San Bruzio

**Gruppi Architettonici**:
- **Gruppo 1**: Facciate RGBW (8 proiettori, controllo sincronizzato)
- **Gruppo 2**: Abside esterno (4 faretti DALI 3000K, solo dimming)
- **Gruppo 3**: Torre campanaria (2 faretti DALI 4000K, solo dimming)

# VINCOLI CROMATICI
- **Evitare**: Colori saturi (rosso/verde/blu puri al 100%) → effetto "Disneyland"
- **Preferire**: Tinte pastello o dominanti cromatiche (es. blu+white 50% per effetto lunare)
- **Proibito**: Strobo, fade rapidi (<5s), animazioni dinamiche
- **Obbligo**: Transizioni graduali minimo 10 secondi

# PALETTE APPROVATA (Soprintendenza)
| Evento | RGB Base | White Mix | Effetto Visivo |
|--------|----------|-----------|----------------|
| Commemorazione | (0, 0, 128) | 80% | Blu notte solenne |
| Natale | (200, 150, 0) | 60% | Oro caldo |
| Pasqua | (255, 240, 200) | 90% | Bianco lunare |
| Festa Patronale | (180, 120, 60) | 70% | Ambra festivo |

# ISTRUZIONI
Analizza l'input JSON contenente:
- `evento`: Nome ricorrenza
- `orarioInizio/Fine`: Durata scenario
- `gruppiArchitettonici`: Array di gruppi disponibili

Genera uno scenario con:
1. **Nome scenario** (max 30 caratteri, senza spazi)
2. **Array di azioni sequenziali** per ogni gruppo
3. **Timing transizioni** (minimo 10s tra stati diversi)
4. **Justification** (riferimenti storici/culturali se pertinenti)

# OUTPUT RICHIESTO
Rispondi SOLO con JSON valido:

{
  "scenarioName": "<stringa_snake_case>",
  "durataStimata": "<HH:MM>",
  "actions": [
    {
      "groupId": <int>,
      "state": {
        "dimLevel": <float 0.0-1.0>,
        "colorRgbw": [<r>, <g>, <b>, <w>],  // valori 0.0-1.0
        "transitionTime": <secondi>
      },
      "timing": {
        "delayFromStart": <secondi>,
        "hold": <secondi>
      }
    }
  ],
  "justification": "<max 200 caratteri>"
}
```

**Input Esempio**:

```json
{
  "context": "scenarioEventoSolennità",
  "evento": "Giorno della Memoria",
  "data": "2025-01-27",
  "orarioInizio": "18:00",
  "orarioFine": "23:00",
  "gruppiArchitettonici": {
    "1": {
      "nome": "Facciate RGBW",
      "tipo": "rgbw",
      "unitIds": ["0x1A", "0x1B", "0x1C"]
    },
    "2": {
      "nome": "Abside",
      "tipo": "dali_dimming",
      "colorTemp": 3000
    }
  },
  "noteStoriche": "Evento di importanza nazionale, richiesta sobrietà cromatica"
}
```

**Output Atteso**:

```json
{
  "scenarioName": "Memoria_Solenne_27Gen",
  "durataStimata": "05:00",
  "actions": [
    {
      "groupId": 1,
      "state": {
        "dimLevel": 0.95,
        "colorRgbw": [0.0, 0.0, 0.4, 0.85],
        "transitionTime": 30
      },
      "timing": {
        "delayFromStart": 0,
        "hold": 18000
      }
    },
    {
      "groupId": 2,
      "state": {
        "dimLevel": 0.80,
        "colorTemp": 3000,
        "transitionTime": 30
      },
      "timing": {
        "delayFromStart": 0,
        "hold": 18000
      }
    }
  ],
  "justification": "Blu profondo (40%) + White dominante per solennità istituzionale. Transizioni lente (30s) per riflessione. Abside in bianco caldo di supporto (CRI 90)."
}
```

-----

### 4.3 Prompt: Diagnostica Predittiva

**Contesto**: Analisi cronologia guasti e anomalie per prevenzione manutenzione.

```markdown
# RUOLO
Sei un ingegnere di manutenzione predittiva specializzato in sistemi DALI e illuminazione LED. Analizzi pattern di guasto per anticipare interventi manutentivi.

# INPUT DATI
Riceverai un JSON contenente:
- **Cronologia energetica**: consumi ultimi 30 giorni per unità
- **Eventi offline**: timestamp e durata disconnessioni
- **Parametri ambientali**: temperatura aria, umidità (se disponibili)
- **Età installazione**: giorni dall'attivazione

# MODELLI DIAGNOSTICI
1. **Degrado LED**: riduzione lumen >15% in 6 mesi → sostituzione preventiva
2. **Instabilità DALI**: offline/online >3 volte/settimana → verifica cablaggio
3. **Anomalia termica**: temperatura driver >75°C → verifica ventilazione
4. **Pattern stagionale**: incremento guasti in mesi umidi → sigillatura IP

# OUTPUT RICHIESTO
{
  "alertLevel": "low|medium|high|critical",
  "affectedUnits": ["<unitId>", ...],
  "diagnosisType": "degrado_led|cablaggio|termica|altro",
  "probabilitàGuastoImminente": <percentuale>,
  "azioneRaccomandata": "<descrizione>",
  "urgenza": "<giorni_prima_intervento>",
  "costoStimato": "<EUR>"
}
```

-----

## 🚀 Deployment

### Installazione Server Comunale

```bash
# 1. Clone repository
git clone https://github.com/comune-magliano/smart-lighting.git
cd smart-lighting

# 2. Configura variabili d'ambiente
cp .env.example .env
nano .env  # Inserire credenziali Casambi, chiavi Cloudflare, etc.

# 3. Avvia stack Docker
docker-compose up -d

# 4. Verifica Ollama
docker exec ollama-magliano ollama list

# 5. Importa workflow n8n
# Accesso: https://localhost:5678
# Import: workflows/*.json
```

### Configurazione ZeroTier

```bash
# Su server
curl -s https://install.zerotier.com | sudo bash
sudo zerotier-cli join <NETWORK_ID>

# Autorizzare su ZeroTier Central
# Configurare firewall: porta 5678 solo su interfaccia zt*
```

-----

## 🔒 Sicurezza

### Threat Model

|Minaccia                      |Mitigazione                        |Priorità|
|------------------------------|-----------------------------------|--------|
|Accesso non autorizzato n8n   |ZeroTier ACL + password strong     |ALTA    |
|Esfiltrazione dati storici    |Encryption-at-rest PostgreSQL      |ALTA    |
|Manipolazione comandi lighting|Firma JWT webhook + rate limiting  |MEDIA   |
|DDoS su Cloudflare Tunnel     |Cloudflare WAF + IP whitelist      |MEDIA   |
|Compromissione container      |Docker rootless + AppArmor profiles|ALTA    |

### Audit Log

Tutti i comandi di illuminazione vengono registrati:

```sql
CREATE TABLE audit_lighting (
  id SERIAL PRIMARY KEY,
  timestamp TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  action VARCHAR(50) NOT NULL,
  user_id VARCHAR(100),
  unit_id VARCHAR(20),
  old_state JSONB,
  new_state JSONB,
  ai_justification TEXT,
  source VARCHAR(20) -- 'manual', 'ai_scheduled', 'ai_emergency'
);
```

-----

## 📊 Monitoraggio

### Dashboard Grafana (Opzionale)

Metriche esposte da n8n e Ollama:

- Latenza chiamate Ollama (p50, p95, p99)
- Uptime unità DALI per gruppo
- Energia cumulativa per Bene Storico
- Contatore scenari attivati (manuale vs. IA)

-----

## 📄 Licenza

Questo progetto è rilasciato sotto licenza **MIT** per la componente software.  
I dati di configurazione specifica (credenziali, topologia rete) rimangono proprietà del Comune di Magliano in Toscana.

-----

## 👥 Contatti

- **Ufficio Tecnico Comunale**: tecnico@comune.magliano.gr.it
- **Fornitore Illuminazione**: Domolight Srl
- **Supporto Casambi**: support@casambi.com

-----

**Versione Documento**: 1.2.0  
**Data Ultimo Aggiornamento**: 27 Ottobre 2025  
**Autore**: Ufficio Innovazione Digitale - Comune di Magliano in Toscana​​​​​​​​​​​​​​​​
