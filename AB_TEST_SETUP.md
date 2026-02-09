# A/B Test Setup Guide - Landing Page Redesign

## Struttura URL

| Variante | URL | Descrizione |
|----------|-----|-------------|
| **A** (attuale) | `https://unflat.finance/` | Landing page Webflow originale |
| **B** (nuova) | `https://unflat.finance/b/` | Landing page React/Vite (riccardo_landing) |

---

## 1. Configurazione Google Tag Manager (GTM-K9J9WXDR)

### 1.1 Creare una variabile Data Layer per la variante

1. In GTM vai a **Variables > User-Defined Variables > New**
2. Tipo: **Data Layer Variable**
3. Nome variabile: `ab_test_variant`
4. Data Layer Variable Name: `ab_test_variant`
5. Salva come: `DLV - AB Test Variant`

### 1.2 Creare una variabile Data Layer per il nome del test

1. **Variables > User-Defined Variables > New**
2. Tipo: **Data Layer Variable**
3. Nome: `ab_test_name`
4. Data Layer Variable Name: `ab_test_name`
5. Salva come: `DLV - AB Test Name`

### 1.3 Creare un Trigger per l'evento A/B

1. **Triggers > New**
2. Tipo: **Custom Event**
3. Event name: `ab_test_variant`
4. Salva come: `CE - AB Test Variant`

### 1.4 Creare un Tag GA4 Event per tracciare la variante

1. **Tags > New**
2. Tipo: **Google Analytics: GA4 Event**
3. Measurement ID: `G-QT9ZNTFJV1`
4. Event Name: `ab_test_impression`
5. Event Parameters:
   - `ab_test_name` = `{{DLV - AB Test Name}}`
   - `ab_test_variant` = `{{DLV - AB Test Variant}}`
6. Trigger: `CE - AB Test Variant`
7. Salva come: `GA4 - AB Test Impression`

---

## 2. Configurazione Google Analytics 4 (G-QT9ZNTFJV1)

### 2.1 Creare dimensioni personalizzate

1. In GA4 vai a **Admin > Custom definitions > Create custom dimension**
2. Crea due dimensioni:
   - **Dimension name**: `AB Test Name` | Scope: Event | Event parameter: `ab_test_name`
   - **Dimension name**: `AB Test Variant` | Scope: Event | Event parameter: `ab_test_variant`

### 2.2 Creare un report di confronto

1. Vai a **Explore > Blank exploration**
2. Aggiungi dimensione: `AB Test Variant`
3. Aggiungi metriche:
   - `Sessions`
   - `Engagement rate`
   - `Conversions` (evento `get_early_access`)
   - `Average engagement time`
4. Filtra per `AB Test Name` = `landing_page_redesign`

### 2.3 Creare un segmento per variante

1. In Explore, crea due segmenti:
   - **Variante A**: Event segment dove `ab_test_variant` = `A`
   - **Variante B**: Event segment dove `ab_test_variant` = `B`

---

## 3. Distribuzione del traffico

Poiché stai usando URL separati, hai diverse opzioni per distribuire il traffico:

### Opzione 1: Google Ads (se usi campagne a pagamento)
- Crea due ad che puntano rispettivamente a `/` e `/b/`
- Distribuisci il budget 50/50

### Opzione 2: GTM Redirect (split automatico)
Aggiungi questo tag in GTM per fare uno split 50/50 automatico:

1. **Tags > New > Custom HTML**
2. Trigger: **Page View** (solo su pagina `/`)
3. HTML:
```html
<script>
  // Split 50/50 - solo per nuovi visitatori
  if (!document.cookie.includes('ab_assigned')) {
    var variant = Math.random() < 0.5 ? 'A' : 'B';
    document.cookie = 'ab_assigned=' + variant + ';path=/;max-age=2592000'; // 30 giorni
    if (variant === 'B') {
      window.location.replace('/b/');
    }
  } else if (document.cookie.includes('ab_assigned=B') && window.location.pathname === '/') {
    window.location.replace('/b/');
  }
</script>
```

### Opzione 3: Link diretti
- Condividi `/` ad un gruppo e `/b/` ad un altro
- Utile per test su audience specifiche (es. newsletter vs social)

---

## 4. Eventi da tracciare (KPIs)

Entrambe le landing page già tracciano:

| Evento | Descrizione | Landing A | Landing B |
|--------|-------------|-----------|-----------|
| `ab_test_impression` | Visualizzazione della variante | ✅ | ✅ |
| `get_early_access` | Click su CTA "Get Early Access" | ✅ | ✅ (via GTM) |
| `page_view` | Visualizzazione pagina (automatico GA4) | ✅ | ✅ |

### Metriche da confrontare:
- **Conversion Rate**: `get_early_access` / `sessions`
- **Engagement Rate**: % sessioni engaged
- **Bounce Rate**: % sessioni con singola page view
- **Avg. Engagement Time**: tempo medio sulla pagina

---

## 5. Durata consigliata del test

- **Minimo**: 2 settimane (per coprire variazioni giornaliere)
- **Campione minimo**: ~1000 sessioni per variante per significatività statistica
- **Tool per calcolo**: https://www.evanmiller.org/ab-testing/sample-size.html

---

## 6. Note tecniche

- Entrambe le landing usano lo stesso GTM container (`GTM-K9J9WXDR`) e GA4 property (`G-QT9ZNTFJV1`)
- Il dataLayer push per l'identificazione della variante avviene prima del caricamento di GTM
- La landing B è una SPA React — gli eventi vengono tracciati via `react-gtm-module`
- Gli asset della landing B sono sotto `/b/assets/`, `/b/images/`, `/b/fonts/`
- Il `vercel.json` include rewrites e caching headers per la sottocartella `/b/`

---

## 7. Checklist pre-lancio

- [ ] Pubblicare le modifiche in GTM (Preview mode prima, poi Publish)
- [ ] Verificare in GA4 DebugView che gli eventi `ab_test_impression` arrivano con i parametri corretti
- [ ] Testare entrambi gli URL: `unflat.finance/` e `unflat.finance/b/`
- [ ] Verificare che le custom dimensions appaiano nei report
- [ ] Impostare il metodo di distribuzione del traffico scelto
- [ ] Deploy su Vercel
