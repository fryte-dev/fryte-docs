# Concept: API Endpoint Split

## Ausgangslage

Der Planner ist ein monolithischer Endpoint der alles auf einmal macht: Routing → POI → Simulation → Optimization. In der Praxis haben die meisten Kunden aber schon eine Route (Polyline aus eigenem TMS, Sygic, PTV, etc.). Die wollen nicht "plane mir eine Route" sondern "optimiere meine existierende Route".

Aktuell akzeptiert die API zwei Input-Modi:
- **Polyline** → wird trotzdem durch HERE geroutet (für Elevation, Turn-by-Turn, Tolls)
- **start_pos / stop_pos** → HERE berechnet die Route komplett

Beide Wege landen am Ende im gleichen monolithischen Flow.

## Idee: Aufsplitten nach Wertschöpfungsstufe

Statt 1 Endpoint der alles macht → modulare Endpoints die einzeln nutzbar sind:

### Stufe 1: Route Input (flexibel)

Der Kunde bringt seine Route mit — als Polyline, als Koordinaten-Paar, oder lässt HERE routen. Das ist kein eigener Endpoint, sondern flexibler Input für alles was danach kommt.

**Inputoptionen:**
| Input | Was passiert |
|-------|-------------|
| `polyline` pro Section | Route wird angereichert (Elevation, Tolls) |
| `start_pos` + `stop_pos` | HERE berechnet Truck-Route |
| Beides leer | Fehler |

**Docs-Fokus:** Klarstellen dass man seine eigene Route mitbringen kann. Die API ist nicht nur ein Routenplaner — sie optimiert existierende Routen.

### Stufe 2: Charging Stations entlang der Route

Eigenständiger Wert: "Gib mir alle Ladestationen entlang meiner Route."

**Existiert intern als:** `POST /api/v1/locations/optimization/pois-along-route`

**Standalone Use Cases:**
- Fleetmanager will Stationen auf einer Route sehen, ohne Optimierung
- Fahrer will wissen welche Stationen verfügbar sind
- Dashboard-Anzeige von Stationen entlang geplanter Routen

**Input:** Route (Polyline) + Corridor + Filter (min_kW, connector_type, truck_approved, cpo, bookable)
**Output:** Liste von Stationen mit Position, Power, Entfernung von Route, Buchbarkeit

### Stufe 3: Simulation (SoC-Vorhersage)

"Wie entwickelt sich mein Batteriestand auf dieser Route?"

**Existiert intern als:** `POST /simulation/simulate`

**Standalone Use Cases:**
- Prüfen ob eine Route ohne Laden fahrbar ist
- SoC-Kurve für eine Kundenroute anzeigen
- "Schaffe ich es bis zum nächsten Depot?"

**Input:** Route + Vehicle + Payload + Startzeit
**Output:** SoC-Kurve, Qualification (drivable/not_drivable)

### Stufe 4: Optimization (Ladeplan)

"Wo und wie lange soll ich laden?" — das Full Package, was der Planner heute macht.

**Ist heute:** `POST /planner/v2` (async)

**Input:** Alles aus Stufe 1–3 + Optimierungsparameter
**Output:** Charging Plan mit Stops, Zeiten, Energiemengen, Alternativen, Tolls

---

## Mögliche Endpoint-Struktur

```
# Standalone Endpoints (synchron, einzeln nutzbar)
POST /v2/stations-along-route     → Stufe 2: Stationen finden
POST /v2/simulate                 → Stufe 3: SoC vorhersagen

# Full Optimization (async, wie heute)
POST /planner/v2                  → Stufe 4: Submit
GET  /planner/jobs/{job_id}       → Status
GET  /planner/jobs                → List
GET  /planner/optimizations/{id}/result → Result
```

Routing (Stufe 1) ist kein eigener Endpoint — es ist der flexible Input für alle anderen Stufen.

---

## Was das für die Docs bedeutet

Die Key Concepts auf der Overview-Seite bilden diese Stufen ab:

| Key Concept | API Endpoint | Standalone? |
|-------------|-------------|-------------|
| Routing | Input-Option (Polyline oder Koordinaten) | Nein — ist Input, kein Service |
| POI Data | `POST /v2/stations-along-route` | Ja |
| Simulation | `POST /v2/simulate` | Ja |
| Optimization | `POST /planner/v2` (async flow) | Ja (nutzt alles oben) |

Jedes Key Concept bekommt eine Doku-Seite die erklärt:
1. Was es macht
2. Wie man es standalone nutzt (wenn möglich)
3. Wie es im Full-Optimization-Flow eingebettet ist
4. Wie es in der Response auftaucht

---

## Open Questions

- **Naming:** `stations-along-route` oder `charging-stations`? `simulate` oder `predict-soc`?
- **Auth:** Gleicher API Key für alle Endpoints?
- **Rate Limits:** Eigene Limits pro Endpoint oder shared?
- **Polyline-Format:** Nur HERE Flexible Polyline oder auch Google Encoded Polyline?
- **Simulation standalone:** Braucht man ein Vehicle — woher kommt das? Gleicher vehicle_id Lookup?
