# Sistema de Standings Actualizado en Tiempo Real

## Arquitectura del Sistema

### Flujo de Datos
```
[API Codeforces] → [PHP Backend] → [JSON] → [JavaScript Frontend] → [DOM HTML]
```

**Ventajas de este enfoque:**
- ✅ Separación clara entre backend y frontend
- ✅ El HTML actúa como template (no se recarga la página)
- ✅ PHP solo provee datos en JSON
- ✅ JavaScript maneja toda la lógica de actualización y animación del DOM
- ✅ Experiencia fluida sin recargas de página

---

## Estructura de Archivos

```
Proyectito/
├── index.php                      # Punto de entrada principal
├── assets/
│   ├── JS/
│   │   ├── standings.js           # ← Lógica principal del standings
│   │   └── utils.js               # Funciones auxiliares
│   └── styles/
│       ├── main.css
│       └── standings.css          # ← Estilos y animaciones
├── includes/
│   ├── config.php                 # Configuración general
│   └── header.php / footer.php    # Templates HTML reutilizables
├── Pages/
│   └── standings.php              # ← Página HTML template
└── services/
    ├── codeforces_api.php         # ← Clase para consumir API Codeforces
    └── api_endpoint.php           # ← Endpoint que devuelve JSON al frontend
```

---

## 1. Template HTML (Pages/standings.php)

**Propósito:** Estructura HTML estática que sirve como template. JavaScript lo poblará con datos.

```php
<?php 
include_once '../includes/config.php';
include_once '../includes/header.php'; 
?>

<main class="standings-container">
    <div class="standings-header">
        <h1>Contest Standings</h1>
        <div class="controls">
            <select id="contestId">
                <option value="">Selecciona un contest</option>
                <!-- Se llenará con JS -->
            </select>
            <button id="btnRefresh" class="btn-primary">🔄 Actualizar</button>
            <label class="auto-refresh">
                <input type="checkbox" id="autoRefresh" checked>
                Auto-refresh cada 10s
            </label>
            <span id="lastUpdate" class="last-update"></span>
        </div>
    </div>

    <!-- Tabla de clasificación - Se llena dinámicamente -->
    <div class="table-wrapper">
        <table id="standingsTable" class="standings-table">
            <thead>
                <tr>
                    <th class="rank-col">Rank</th>
                    <th class="user-col">Usuario</th>
                    <th class="points-col">Puntos</th>
                    <!-- Columnas de problemas se generan dinámicamente -->
                    <div id="problemHeaders"></div>
                </tr>
            </thead>
            <tbody id="standingsBody">
                <!-- Filas se generan con JavaScript -->
            </tbody>
        </table>
    </div>

    <!-- Loading indicator -->
    <div id="loadingOverlay" class="loading-overlay hidden">
        <div class="spinner"></div>
        <p>Cargando datos...</p>
    </div>
</main>

<!-- Cargar JavaScript -->
<script src="<?php echo BASE_URL; ?>assets/JS/standings.js" type="module"></script>

<?php include_once '../includes/footer.php'; ?>
```

---

## 2. Backend PHP - API Endpoint (services/api_endpoint.php)

**Propósito:** Recibe peticiones AJAX, consulta Codeforces API, procesa datos y devuelve JSON.

```php
<?php
header('Content-Type: application/json');
header('Access-Control-Allow-Origin: *');

require_once '../includes/config.php';
require_once 'codeforces_api.php';

// Parsear acción solicitada
$action = $_GET['action'] ?? '';

try {
    $api = new CodeforcesAPI();
    
    switch ($action) {
        case 'getContests':
            // Obtener lista de contests disponibles
            $contests = $api->getContests();
            sendResponse(true, $contests);
            break;
            
        case 'getStandings':
            // Obtener standings de un contest específico
            $contestId = $_GET['contestId'] ?? null;
            
            if (!$contestId) {
                sendResponse(false, null, 'Contest ID requerido');
            }
            
            $standings = $api->getContestStandings($contestId);
            sendResponse(true, $standings);
            break;
            
        default:
            sendResponse(false, null, 'Acción no válida');
    }
    
} catch (Exception $e) {
    sendResponse(false, null, $e->getMessage());
}

/**
 * Envía respuesta JSON estandarizada
 */
function sendResponse($success, $data = null, $error = null) {
    echo json_encode([
        'success' => $success,
        'data' => $data,
        'error' => $error,
        'timestamp' => time()
    ]);
    exit;
}
```

---

## 3. Servicio Codeforces API (services/codeforces_api.php)

**Propósito:** Encapsula la lógica de comunicación con la API de Codeforces.

```php
<?php

class CodeforcesAPI {
    private $baseUrl = 'https://codeforces.com/api/';
    private $cacheDir = __DIR__ . '/../cache/';
    private $cacheTTL = 300; // 5 minutos
    
    public function __construct() {
        // Crear directorio de caché si no existe
        if (!is_dir($this->cacheDir)) {
            mkdir($this->cacheDir, 0755, true);
        }
    }
    
    /**
     * Obtiene lista de contests
     */
    public function getContests() {
        $endpoint = 'contest.list';
        $data = $this->makeRequest($endpoint);
        
        if ($data['status'] !== 'OK') {
            throw new Exception('Error al obtener contests');
        }
        
        // Filtrar solo contests activos o recientes
        $contests = array_filter($data['result'], function($contest) {
            return $contest['phase'] !== 'FINISHED';
        });
        
        return array_values($contests);
    }
    
    /**
     * Obtiene standings de un contest
     */
    public function getContestStandings($contestId, $from = 1, $count = 100) {
        $endpoint = 'contest.standings';
        $params = [
            'contestId' => $contestId,
            'from' => $from,
            'count' => $count,
            'showUnofficial' => false
        ];
        
        $data = $this->makeRequest($endpoint, $params);
        
        if ($data['status'] !== 'OK') {
            throw new Exception('Error al obtener standings');
        }
        
        return $this->processStandings($data['result']);
    }
    
    /**
     * Procesa y formatea los standings
     */
    private function processStandings($result) {
        $problems = $result['problems'];
        $rows = $result['rows'];
        
        $standings = [
            'problems' => array_map(function($p) {
                return [
                    'index' => $p['index'],
                    'name' => $p['name'],
                    'points' => $p['points'] ?? 0
                ];
            }, $problems),
            'rows' => []
        ];
        
        foreach ($rows as $row) {
            $party = $row['party'];
            $problemResults = $row['problemResults'];
            
            $standings['rows'][] = [
                'rank' => $row['rank'],
                'handle' => $party['members'][0]['handle'] ?? 'Unknown',
                'points' => $row['points'],
                'penalty' => $row['penalty'],
                'problems' => array_map(function($pr) {
                    return [
                        'points' => $pr['points'],
                        'rejectedAttempts' => $pr['rejectedAttemptCount'],
                        'verdict' => $pr['bestSubmissionTimeSeconds'] > 0 ? 'ACCEPTED' : 
                                    ($pr['rejectedAttemptCount'] > 0 ? 'WRONG_ANSWER' : 'PENDING'),
                        'time' => $pr['bestSubmissionTimeSeconds'] ?? 0
                    ];
                }, $problemResults)
            ];
        }
        
        return $standings;
    }
    
    /**
     * Realiza petición HTTP a la API
     */
    private function makeRequest($endpoint, $params = []) {
        // Construir cache key
        $cacheKey = md5($endpoint . serialize($params));
        $cacheFile = $this->cacheDir . $cacheKey . '.json';
        
        // Verificar caché
        if (file_exists($cacheFile) && (time() - filemtime($cacheFile)) < $this->cacheTTL) {
            return json_decode(file_get_contents($cacheFile), true);
        }
        
        // Hacer petición a la API
        $url = $this->baseUrl . $endpoint;
        if (!empty($params)) {
            $url .= '?' . http_build_query($params);
        }
        
        $ch = curl_init();
        curl_setopt($ch, CURLOPT_URL, $url);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($ch, CURLOPT_TIMEOUT, 15);
        curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, false);
        
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        
        if ($httpCode !== 200) {
            throw new Exception("Error HTTP: $httpCode");
        }
        
        $data = json_decode($response, true);
        
        // Guardar en caché
        file_put_contents($cacheFile, $response);
        
        return $data;
    }
}
```

---

## 4. JavaScript Frontend (assets/JS/standings.js)

**Propósito:** Maneja toda la lógica de UI, actualización de datos y animaciones.

```javascript
/**
 * Sistema de Standings con actualización en tiempo real
 */
class StandingsManager {
    constructor() {
        // Estado del sistema
        this.currentContestId = null;
        this.currentStandings = [];
        this.previousRanks = new Map(); // Para tracking de cambios de posición
        this.autoRefreshInterval = null;
        this.isLoading = false;
        
        // Elementos DOM
        this.contestSelect = document.getElementById('contestId');
        this.standingsTable = document.getElementById('standingsTable');
        this.standingsBody = document.getElementById('standingsBody');
        this.btnRefresh = document.getElementById('btnRefresh');
        this.autoRefreshCheckbox = document.getElementById('autoRefresh');
        this.loadingOverlay = document.getElementById('loadingOverlay');
        this.lastUpdateSpan = document.getElementById('lastUpdate');
        
        // Configuración
        this.API_BASE = 'services/api_endpoint.php';
        this.REFRESH_INTERVAL = 10000; // 10 segundos
        
        this.init();
    }
    
    /**
     * Inicializa el sistema
     */
    init() {
        this.loadContests();
        this.setupEventListeners();
        
        // Iniciar auto-refresh si está activado
        if (this.autoRefreshCheckbox.checked) {
            this.startAutoRefresh();
        }
    }
    
    /**
     * Configurar event listeners
     */
    setupEventListeners() {
        // Cambio de contest
        this.contestSelect.addEventListener('change', (e) => {
            this.currentContestId = e.target.value;
            if (this.currentContestId) {
                this.loadStandings();
            }
        });
        
        // Botón de refresh manual
        this.btnRefresh.addEventListener('click', () => {
            if (this.currentContestId) {
                this.loadStandings();
            }
        });
        
        // Toggle auto-refresh
        this.autoRefreshCheckbox.addEventListener('change', (e) => {
            if (e.target.checked) {
                this.startAutoRefresh();
            } else {
                this.stopAutoRefresh();
            }
        });
    }
    
    /**
     * Carga lista de contests disponibles
     */
    async loadContests() {
        try {
            const response = await fetch(`${this.API_BASE}?action=getContests`);
            const data = await response.json();
            
            if (data.success) {
                this.populateContestSelect(data.data);
            } else {
                this.showError('Error al cargar contests: ' + data.error);
            }
        } catch (error) {
            this.showError('Error de red: ' + error.message);
        }
    }
    
    /**
     * Llena el selector de contests
     */
    populateContestSelect(contests) {
        // Limpiar opciones existentes excepto la primera
        this.contestSelect.innerHTML = '<option value="">Selecciona un contest</option>';
        
        contests.forEach(contest => {
            const option = document.createElement('option');
            option.value = contest.id;
            option.textContent = `${contest.name} (ID: ${contest.id})`;
            this.contestSelect.appendChild(option);
        });
    }
    
    /**
     * Carga standings del contest seleccionado
     */
    async loadStandings() {
        if (this.isLoading) return;
        
        this.isLoading = true;
        this.showLoading(true);
        
        try {
            const response = await fetch(
                `${this.API_BASE}?action=getStandings&contestId=${this.currentContestId}`
            );
            const data = await response.json();
            
            if (data.success) {
                this.updateStandings(data.data);
                this.updateLastUpdateTime(data.timestamp);
            } else {
                this.showError('Error: ' + data.error);
            }
        } catch (error) {
            this.showError('Error al cargar standings: ' + error.message);
        } finally {
            this.isLoading = false;
            this.showLoading(false);
        }
    }
    
    /**
     * Actualiza la tabla de standings con animaciones
     */
    updateStandings(standings) {
        const { problems, rows } = standings;
        
        // Actualizar headers de problemas
        this.updateProblemHeaders(problems);
        
        // Guardar rankings antiguos para detectar cambios
        const oldRanks = new Map(this.previousRanks);
        this.previousRanks.clear();
        
        // Limpiar tabla
        this.standingsBody.innerHTML = '';
        
        // Crear filas
        rows.forEach((row, index) => {
            const tr = this.createStandingRow(row, oldRanks);
            this.standingsBody.appendChild(tr);
            this.previousRanks.set(row.handle, row.rank);
        });
        
        // Guardar standings actuales
        this.currentStandings = rows;
    }
    
    /**
     * Actualiza headers de problemas en la tabla
     */
    updateProblemHeaders(problems) {
        const thead = this.standingsTable.querySelector('thead tr');
        
        // Remover headers antiguos de problemas
        const oldHeaders = thead.querySelectorAll('.problem-header');
        oldHeaders.forEach(h => h.remove());
        
        // Agregar nuevos headers
        problems.forEach(problem => {
            const th = document.createElement('th');
            th.className = 'problem-header';
            th.textContent = problem.index;
            th.title = problem.name;
            thead.appendChild(th);
        });
    }
    
    /**
     * Crea una fila de la tabla con animaciones
     */
    createStandingRow(row, oldRanks) {
        const tr = document.createElement('tr');
        tr.dataset.handle = row.handle;
        
        // Detectar cambio de posición
        const oldRank = oldRanks.get(row.handle);
        let rankChangeClass = '';
        
        if (oldRank !== undefined && oldRank !== row.rank) {
            if (oldRank > row.rank) {
                rankChangeClass = 'rank-up';  // Subió posiciones
            } else {
                rankChangeClass = 'rank-down'; // Bajó posiciones
            }
        }
        
        if (rankChangeClass) {
            tr.classList.add(rankChangeClass);
            // Remover clase después de animación
            setTimeout(() => tr.classList.remove(rankChangeClass), 2000);
        }
        
        // Crear celdas
        tr.innerHTML = `
            <td class="rank-cell">${row.rank}</td>
            <td class="user-cell">${row.handle}</td>
            <td class="points-cell">${row.points}</td>
            ${this.createProblemCells(row.problems)}
        `;
        
        return tr;
    }
    
    /**
     * Crea celdas de problemas con estados
     */
    createProblemCells(problems) {
        return problems.map(problem => {
            const verdict = problem.verdict;
            let cellClass = 'problem-cell';
            let content = '-';
            
            if (verdict === 'ACCEPTED') {
                cellClass += ' accepted';
                content = `
                    <div class="verdict-badge success">✓</div>
                    <div class="problem-time">${this.formatTime(problem.time)}</div>
                `;
            } else if (verdict === 'WRONG_ANSWER') {
                cellClass += ' wrong-answer';
                content = `
                    <div class="verdict-badge error">✗</div>
                    <div class="attempts">-${problem.rejectedAttempts}</div>
                `;
            } else if (problem.rejectedAttempts > 0) {
                cellClass += ' pending';
                content = `<div class="attempts">-${problem.rejectedAttempts}</div>`;
            }
            
            return `<td class="${cellClass}">${content}</td>`;
        }).join('');
    }
    
    /**
     * Formatea tiempo en segundos a mm:ss
     */
    formatTime(seconds) {
        const mins = Math.floor(seconds / 60);
        const secs = seconds % 60;
        return `${mins}:${secs.toString().padStart(2, '0')}`;
    }
    
    /**
     * Inicia auto-refresh
     */
    startAutoRefresh() {
        this.stopAutoRefresh(); // Limpiar cualquier intervalo existente
        
        this.autoRefreshInterval = setInterval(() => {
            if (this.currentContestId && !this.isLoading) {
                this.loadStandings();
            }
        }, this.REFRESH_INTERVAL);
    }
    
    /**
     * Detiene auto-refresh
     */
    stopAutoRefresh() {
        if (this.autoRefreshInterval) {
            clearInterval(this.autoRefreshInterval);
            this.autoRefreshInterval = null;
        }
    }
    
    /**
     * Muestra/oculta loading overlay
     */
    showLoading(show) {
        this.loadingOverlay.classList.toggle('hidden', !show);
    }
    
    /**
     * Actualiza tiempo de última actualización
     */
    updateLastUpdateTime(timestamp) {
        const date = new Date(timestamp * 1000);
        this.lastUpdateSpan.textContent = `Última actualización: ${date.toLocaleTimeString()}`;
    }
    
    /**
     * Muestra mensaje de error
     */
    showError(message) {
        console.error(message);
        // Podrías implementar un toast o modal aquí
        alert(message);
    }
}

// Inicializar cuando el DOM esté listo
document.addEventListener('DOMContentLoaded', () => {
    const standings = new StandingsManager();
});
```

---

## 5. CSS con Animaciones (assets/styles/standings.css)

```css
/* Container principal */
.standings-container {
    max-width: 1400px;
    margin: 0 auto;
    padding: 20px;
}

/* Header y controles */
.standings-header {
    margin-bottom: 20px;
}

.controls {
    display: flex;
    gap: 15px;
    align-items: center;
    flex-wrap: wrap;
    margin-top: 15px;
}

.controls select {
    padding: 8px 12px;
    border: 1px solid #ddd;
    border-radius: 4px;
    min-width: 200px;
}

.btn-primary {
    padding: 8px 16px;
    background: #007bff;
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    transition: background 0.3s;
}

.btn-primary:hover {
    background: #0056b3;
}

.last-update {
    color: #666;
    font-size: 0.9em;
}

/* Tabla de standings */
.table-wrapper {
    overflow-x: auto;
    box-shadow: 0 2px 4px rgba(0,0,0,0.1);
    border-radius: 8px;
}

.standings-table {
    width: 100%;
    border-collapse: collapse;
    background: white;
    font-family: 'Courier New', monospace;
}

.standings-table thead {
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    color: white;
    position: sticky;
    top: 0;
    z-index: 10;
}

.standings-table th {
    padding: 12px 8px;
    text-align: center;
    font-weight: 600;
    border: 1px solid rgba(255,255,255,0.2);
}

.standings-table td {
    padding: 10px 8px;
    text-align: center;
    border: 1px solid #e0e0e0;
}

.standings-table tbody tr {
    transition: all 0.5s ease;
}

.standings-table tbody tr:hover {
    background-color: #f5f5f5;
}

/* Columnas específicas */
.rank-cell {
    font-weight: bold;
    color: #333;
    min-width: 60px;
}

.user-cell {
    text-align: left !important;
    font-weight: 500;
    min-width: 150px;
}

.points-cell {
    font-weight: bold;
    color: #2e7d32;
    min-width: 80px;
}

/* Celdas de problemas */
.problem-cell {
    min-width: 80px;
    position: relative;
    font-size: 0.85em;
}

.problem-cell.accepted {
    background-color: rgba(76, 175, 80, 0.15);
    border-left: 3px solid #4caf50;
}

.problem-cell.wrong-answer {
    background-color: rgba(244, 67, 54, 0.15);
    border-left: 3px solid #f44336;
}

.problem-cell.pending {
    background-color: rgba(255, 152, 0, 0.1);
}

.verdict-badge {
    font-size: 1.2em;
    font-weight: bold;
}

.verdict-badge.success {
    color: #4caf50;
}

.verdict-badge.error {
    color: #f44336;
}

.problem-time {
    font-size: 0.8em;
    color: #666;
    margin-top: 2px;
}

.attempts {
    color: #f57c00;
    font-weight: bold;
}

/* ANIMACIONES DE RANKING */

/* Subió de posición - Verde */
@keyframes rankUpAnimation {
    0% {
        background-color: rgba(76, 175, 80, 0.4);
        transform: translateY(10px);
    }
    50% {
        background-color: rgba(76, 175, 80, 0.6);
    }
    100% {
        background-color: transparent;
        transform: translateY(0);
    }
}

.rank-up {
    animation: rankUpAnimation 2s ease-out;
    border-left: 4px solid #4caf50;
}

/* Bajó de posición - Rojo */
@keyframes rankDownAnimation {
    0% {
        background-color: rgba(244, 67, 54, 0.4);
        transform: translateY(-10px);
    }
    50% {
        background-color: rgba(244, 67, 54, 0.6);
    }
    100% {
        background-color: transparent;
        transform: translateY(0);
    }
}

.rank-down {
    animation: rankDownAnimation 2s ease-out;
    border-left: 4px solid #f44336;
}

/* Loading overlay */
.loading-overlay {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background: rgba(255, 255, 255, 0.9);
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    z-index: 9999;
    transition: opacity 0.3s;
}

.loading-overlay.hidden {
    display: none;
}

.spinner {
    width: 50px;
    height: 50px;
    border: 4px solid #f3f3f3;
    border-top: 4px solid #667eea;
    border-radius: 50%;
    animation: spin 1s linear infinite;
}

@keyframes spin {
    0% { transform: rotate(0deg); }
    100% { transform: rotate(360deg); }
}

/* Responsive */
@media (max-width: 768px) {
    .controls {
        flex-direction: column;
        align-items: stretch;
    }
    
    .controls select,
    .btn-primary {
        width: 100%;
    }
    
    .standings-table {
        font-size: 0.85em;
    }
    
    .standings-table th,
    .standings-table td {
        padding: 6px 4px;
    }
}
```

---

## 6. Configuración (includes/config.php)

```php
<?php
// Definir URL base
$protocol = isset($_SERVER['HTTPS']) && $_SERVER['HTTPS'] === 'on' ? 'https://' : 'http://';
$host = $_SERVER['HTTP_HOST'];
$path = dirname($_SERVER['SCRIPT_NAME']);
$path = $path === '\\' || $path === '/' ? '' : $path;

define('BASE_URL', $protocol . $host . $path . '/');
define('BASE_PATH', __DIR__ . '/../');

// Configuración de errores (desarrollo)
ini_set('display_errors', 1);
error_reporting(E_ALL);

// Configuración de timezone
date_default_timezone_set('America/Mexico_City');
```

---

## Resumen del Flujo

### Carga Inicial:
1. Usuario abre `Pages/standings.php` (HTML template)
2. JavaScript (`standings.js`) se carga y ejecuta
3. JS hace fetch a `services/api_endpoint.php?action=getContests`
4. PHP consulta Codeforces API y devuelve JSON
5. JS puebla el selector de contests

### Actualización de Standings:
1. Usuario selecciona un contest
2. JS hace fetch a `services/api_endpoint.php?action=getStandings&contestId=XXX`
3. PHP consulta Codeforces API, procesa datos y devuelve JSON
4. JS recibe datos y actualiza el DOM:
   - Compara rankings antiguos vs nuevos
   - Aplica clases CSS para animaciones (rank-up/rank-down)
   - Actualiza estado de problemas (ACCEPTED/WRONG_ANSWER)
5. Auto-refresh se ejecuta cada 10 segundos si está activado

### Animaciones CSS:
- Filas que suben: fondo verde + animación hacia arriba
- Filas que bajan: fondo rojo + animación hacia abajo
- Problemas accepted: fondo verde claro + ✓
- Problemas wrong: fondo rojo claro + ✗

---

## Próximos Pasos

1. ✅ Implementar autenticación con Codeforces API (si es necesaria)
2. ✅ Mejorar sistema de caché para reducir llamadas a la API
3. ✅ Agregar filtros (por usuario, por problema)
4. ✅ Implementar WebSockets para actualizaciones en tiempo real
5. ✅ Agregar gráficos de progreso por problema
6. ✅ Implementar modo oscuro

---

## Ventajas de esta Arquitectura

✅ **Separación de responsabilidades**: PHP solo maneja datos, JS maneja UI
✅ **Sin recargas de página**: Mejor UX con actualizaciones fluidas
✅ **Reutilizable**: El endpoint PHP puede usarse para otras vistas
✅ **Escalable**: Fácil agregar nuevas funcionalidades
✅ **Performance**: Sistema de caché reduce carga en API externa
✅ **Animaciones suaves**: CSS transitions y animations
✅ **Responsive**: Funciona en móviles y tablets

