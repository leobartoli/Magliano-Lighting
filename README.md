Capito. Il progetto riguarda l'illuminazione di beni storici (San Bruzio e Mura), il che eleva l'importanza della precisione degli scenari. Inoltre, la sicurezza e la sovranità del dato impongono l'uso di un Agente IA locale tramite Ollama su Docker, anziché servizi cloud esterni.
Riscrivo il README.md per GitHub, aggiornando l'architettura IA e i Prompt per riflettere la nuova enfasi sulla storicità e l'esecuzione IA locale.
💡 Smart Lighting Management - Comune di Magliano in Toscana
Questo repository documenta l'architettura di gestione centralizzata e intelligente dell'illuminazione pubblica per i Beni Storici (San Bruzio e le Mura). Il sistema utilizza un'infrastruttura Zero Trust con Ollama per l'intelligenza artificiale locale e l'integrazione Cloud Casambi.
1. 🏗️ Architettura del Sistema: Sovranità del Dato e IA Locale
La configurazione è progettata per mantenere l'analisi dei dati sensibili (diagnostica e logica degli scenari) all'interno dell'infrastruttura comunale.
1.1 Stack Infrastrutturale (Livello 2: Server Comunale)
| Componente | Ruolo | Protocollo/Tecnologia |
|---|---|---|
| Docker | Containerizzazione | Ambiente isolato per n8n e Ollama. |
| n8n | Motore di Orchestrazione | Esegue i workflow, gestisce le API Casambi e comunica con Ollama. |
| Ollama | Agente IA Locale | Esegue modelli di linguaggio (LLM) per la diagnostica predittiva e la generazione di scenari luminosi. |
| ZeroTier | Rete Privata (VPN) | Garantisce l'accesso amministrativo sicuro (Zero Trust) al server, Docker e all'interfaccia n8n. |
| Cloudflare Tunnel | Accesso Pubblico Sicuro | Espone l'interfaccia n8n e i Webhook su Internet in modo sicuro. |
1.2 Stack Illuminazione (Livello 1: Beni Storici)
La soluzione DALI/Casambi funge da base fisica per l'attuazione dei comandi IA.
| Componente | Fonte | Ruolo nell'Integrazione |
|---|---|---|
| Corpi Illuminanti | [cite_start]Preventivo Domolight [cite: 2] | [cite_start]Apparecchi DALI per controllo individuale (incluse versioni RGBW speciali)[cite: 2]. |
| Salvador 1064 | [cite_start]Preventivo Domolight [cite: 3] | [cite_start]Controller DALI-Casambi[cite: 3]. |
| Casambi Cloud Gateway | [cite_start]Preventivo Domolight [cite: 3] | [cite_start]Ponte di connettività al Cloud Casambi per il controllo remoto e l'integrazione con le Mura[cite: 3]. |
2. 🔌 Flusso di Integrazione Casambi in n8n
[cite_start]Il sistema utilizza la connettività di rete del Router 4G [cite: 3] per l'accesso al Casambi Cloud.
2.1 HTTP/REST: Autenticazione e Dati per l'IA
Il nodo HTTP Request di n8n esegue l'autenticazione e la raccolta di dati storici e diagnostici.
 * Autenticazione: POST a /v1/networks/userlogin per ottenere la sessionId.
 * Raccolta Dati DALI: GET a /v1/networks/{networkId}/datapoints per recuperare i log energetici e diagnostici, essenziali per alimentare il modello di diagnostica predittiva in Ollama.
2.2 WebSocket: Comando in Tempo Reale e Monitoraggio
Il nodo WebSocket di n8n mantiene la connessione aperta per il controllo istantaneo.
| Ruolo | Azione n8n | Scopo |
|---|---|---|
| Invio Comando IA | n8n invia un messaggio JSON sulla connessione WebSocket attiva. | Esecuzione immediata dei livelli di dimming o degli scenari RGBW decisi da Ollama. |
| Ricezione Evento | n8n Listener riceve UnitChangedEvent in tempo reale. | Attiva immediatamente i workflow di diagnostica se un'unità va offline o segnala un errore. |
3. 🧠 Integrazione Agente IA Locale (Ollama su Docker)
Ollama viene eseguito come un container separato sulla macchina Docker. n8n comunica con Ollama tramite una semplice richiesta HTTP interna alla rete Docker (es. http://ollama-service:11434/api/generate).
3.1 Prompt IA per la Gestione Ordinaria (Beni Storici)
L'attenzione è sulla conservazione del bene e sulla percezione visiva, più che sulla mera efficienza energetica.
Ruolo: Agisci come un consulente di conservazione dei beni culturali e lighting designer. L'obiettivo è garantire la massima leggibilità architettonica ed evitare stress termici o di esposizione per l'edificio storico.
Istruzioni: Calcola il livello di luminosità ottimale (CRI e temperatura colore fissa, dimLevel tra 0.60 e 1.0) per l'illuminazione generale notturna del patrimonio storico. Priorità massima alla sicurezza visiva e alla minimizzazione dell'inquinamento luminoso.
Input n8n:
{
  "context": "dimmingOrdinarioStorico",
  "oraCorrente": "01:30",
  [span_0](start_span)"livelloRiferimento": 0.85, // Livello base definito dai test notturni[span_0](end_span)
  "trafficoPedonale": "Assente",
  "vincoliArchitettonici": "Bassa luminosità necessaria in facciata per preservare l'integrità visiva notturna."
}

Output Atteso (Azione per n8n):
{
  "recommendedAction": "dimming",
  "dimLevel": 0.60, // Riduzione controllata per l'assenza di traffico
  "justification": "Riduzione sicura a 0.60 per bilanciare il risparmio e l'effetto visivo in orario di traffico nullo, in linea con i requisiti di conservazione."
}

3.2 Prompt IA per Scenari Eventi e Ricorrenze
[cite_start]Utilizzato per definire gli scenari architetturali emozionali, come quelli per le facciate RGBW[cite: 2].
Ruolo: Agisci come un direttore artistico per l'illuminazione di un monumento. Traduci il tema dell'evento in parametri RGBW e intensità luminosa, rispettando sempre la dignità storica dell'edificio.
Istruzioni: Definisci i parametri di illuminazione per la "Commemorazione dei Caduti". L'illuminazione deve essere evocativa, mantenendo i proiettori RGBW (Rif. B) sulle facciate con un colore istituzionale. La luce deve essere stabile e solenne.
Input n8n:
{
  "context": "scenarioEventoSolemnità",
  "evento": "Giorno della Memoria/Commemorazione",
  "gruppiArchitettonici": {
    "1": "Facciate RGBW (Rif. B)",
    "2": "Abside Esterno (Rif. C)"
  }
}

Output Atteso (Sequenza di Azioni per n8n):
{
  "scenarioName": "Commemorazione_Solenne",
  "actions": [
    {
      "method": "setGroupState",
      "groupId": 1,
      "state": {
        "dimLevel": 1.0,
        "colorRgbw": [0.0, 0.0, 0.5, 1.0] // Colore Blu profondo (solenne)
      }
    },
    {
      "method": "setGroupState",
      "groupId": 2,
      "state": {
        "dimLevel": 0.90,
        "colorTemp": 3000 // Bianco caldo d'accento
      }
    }
  ]
}

