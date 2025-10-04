<template>
    <div class="assistance-module">
        <!-- Título principal del módulo -->
        <h2>Registro de Asistencia</h2>
        <p>Por favor, ingresa tu código de empleado y selecciona tu acción.</p>
        <!-- Campo para ingresar el ID del empleado -->
        <input v-model="employeeId" type="number" placeholder="Código de Empleado" :disabled="loading"
            @blur="getHistory" />

        <!-- Botones para registrar entrada o salida -->
        <div class="actions">
            <!-- Botón de ENTRADA -->
            <!-- Se desactiva si no hay employeeId o está cargando -->
            <button @click="recordAssistance('entrada')" :disabled="!employeeId || loading" class="btn-check-in">
                <!-- Muestra texto dinámico dependiendo del estado -->
                {{ loading && actionType === 'entrada' ? 'Registrando...' : 'Marcar ENTRADA' }}
            </button>

            <!-- Botón de SALIDA -->
            <button @click="recordAssistance('salida')" :disabled="!employeeId || loading" class="btn-check-out">
                {{ loading && actionType === 'salida' ? 'Registrando...' : 'Marcar SALIDA' }}
            </button>
        </div>

        <!-- Mensaje de estado: éxito o error -->
        <p v-if="message" :class="['status-message', statusType]">
            {{ message }}
        </p>

        <!-- Sección de historial -->
        <!-- Solo se muestra si hay registros en 'history' -->
        <div v-if="history.length > 0" class="history-section">
            <!-- Estado actual del empleado (Entrada o Salida) -->
            <h3>
                Estado Actual:
                <span :class="currentStatus === 'Entrada' ? 'status-in' : 'status-out'">
                    {{ currentStatus }}
                </span>
            </h3>

            <!-- Tabla del historial reciente -->
            <h4>Historial Reciente:</h4>
            <table class="history-table">
                <thead>
                    <tr>
                        <th>Tipo</th>
                        <th>Fecha</th>
                        <th>Hora</th>
                        <th>Acciones</th>
                    </tr>
                </thead>
                <tbody>
                    <!-- Renderiza dinámicamente cada registro de asistencia -->
                    <tr v-for="(record, index) in history" :key="index">
                        <td>
                            <span :class="record.type === 'Entrada' ? 'status-in' : 'status-out'">
                                {{ record.type }}
                            </span>
                        </td>
                        <td>{{ record.fecha }}</td>
                        <td>{{ record.hora }}</td>
                        <td>
                            <button 
                                @click="deleteRecord(record.asistencia_id, index)" 
                                :disabled="deletingRecord === record.asistencia_id"
                                class="btn-delete"
                                title="Eliminar registro"
                            >
                                {{ deletingRecord === record.asistencia_id ? '...' : '🗑️' }}
                            </button>
                        </td>
                    </tr>
                </tbody>
            </table>
        </div>
    </div>
</template>

<script setup>
// Importamos los composables de Vue 3
import { ref, watch } from 'vue';
// Importamos axios para hacer peticiones HTTP
import axios from 'axios';

// URL base del backend, tomada desde las variables de entorno
const API_URL = import.meta.env.VITE_API_BASE_URL;

// ======================
// Variables reactivas
// ======================
const employeeId = ref(null);      // ID del empleado
const loading = ref(false);        // Estado de carga para evitar acciones múltiples
const message = ref('');           // Mensaje de estado (éxito o error)
const statusType = ref('');        // Tipo de mensaje ('success' o 'error')
const actionType = ref('');        // Acción actual ('entrada' o 'salida')
const history = ref([]);           // Historial de asistencias
const currentStatus = ref('desconocido'); // Estado actual del empleado
const deletingRecord = ref(null);     // ID del registro que se está eliminando


// ======================
// Watcher: Observa cambios en employeeId
// ======================
// Si el usuario borra el código, se limpia el historial
watch(employeeId, (newId) => {
    if (!newId) {
        history.value = [];
        currentStatus.value = 'desconocido';
    }
});

// ======================
// Función: Registrar Asistencia
// ======================
const recordAssistance = async (type) => {
    // Validación: si no hay ID, mostramos error
    if (!employeeId.value) {
        message.value = 'Por favor, ingresa tu código de empleado.';
        statusType.value = 'error';
        return;
    }

    // Activamos el estado de carga
    loading.value = true;
    actionType.value = type; // Guardamos la acción actual
    message.value = '';

    // Datos a enviar al backend
    const assistanceData = {
        employeeId: employeeId.value,
        type: type
    };

    try {
        // Petición POST al backend para registrar la asistencia
        const response = await axios.post(`${API_URL}/public/attendances/record`, assistanceData);

        // Si la respuesta es exitosa, mostramos el mensaje y actualizamos historial
        if (response.data.success) {
            message.value = response.data.message;
            statusType.value = 'success';
            await getHistory(); // Refrescamos historial
            employeeId.value = null; // Limpiamos el campo
        }
    } catch (error) {
        // Captura de errores del servidor o de conexión
        console.error('Error al registrar la asistencia:', error);

        // Mostramos mensaje adecuado al usuario
        if (error.response && error.response.data && error.response.data.message) {
            message.value = `ERROR: ${error.response.data.message}`;
        } else {
            message.value = 'Error de conexión. Intenta de nuevo.';
        }
        statusType.value = 'error';
    } finally {
        // Finalizamos el estado de carga
        loading.value = false;
        actionType.value = '';
    }
};


// ======================
// Función: Obtener Historial
// ======================
const getHistory = async () => {
    // Si no hay ID, reiniciamos los valores
    if (!employeeId.value) {
        history.value = [];
        currentStatus.value = 'desconocido';
        return;
    }

    try {
        // Petición GET al backend para obtener el historial del empleado
        const response = await axios.get(`${API_URL}/public/attendances/${employeeId.value}`);

        // Si la respuesta es exitosa, actualizamos el historial
        if (response.data.success) {
            history.value = response.data.history;
            currentStatus.value = response.data.current_status;
        }
    } catch (error) {
        // Si ocurre un error, lo mostramos en consola y limpiamos los datos
        console.error('Error al obtener el historial:', error);
        history.value = [];
        currentStatus.value = 'desconocido';
    }
};


// ======================
// Función: Eliminar Registro de Asistencia
// ======================
const deleteRecord = async (recordId, index) => {
    // Confirmación antes de eliminar
    if (!confirm('¿Estás seguro de que deseas eliminar este registro?')) {
        return;
    }

    // Marcamos el registro como "eliminando"
    deletingRecord.value = recordId;
    message.value = '';

    try {
        // Petición DELETE al backend para eliminar el registro
        const response = await axios.delete(`${API_URL}/public/attendances/delete/${recordId}`);

        // Si la eliminación es exitosa
        if (response.data.success) {
            // Removemos el registro del array local
            history.value.splice(index, 1);
            
            // Actualizamos el estado actual si era el registro más reciente
            if (index === 0 && history.value.length > 0) {
                currentStatus.value = history.value[0].type;
            } else if (history.value.length === 0) {
                currentStatus.value = 'desconocido';
            }

            message.value = 'Registro eliminado exitosamente.';
            statusType.value = 'success';
        }
    } catch (error) {
        // Captura de errores
        console.error('Error al eliminar el registro:', error);
        
        if (error.response && error.response.data && error.response.data.message) {
            message.value = `ERROR: ${error.response.data.message}`;
        } else {
            message.value = 'Error de conexión al eliminar el registro.';
        }
        statusType.value = 'error';
    } finally {
        // Limpiamos el estado de eliminación
        deletingRecord.value = null;
    }
};

</script>

<style scoped>
.assistance-module {
    max-width: 450px;
    margin: 50px auto;
    padding: 25px;
    border: 1px solid #ccc;
    border-radius: 8px;
    text-align: center;
    font-family: sans-serif;
}

input {
    width: 90%;
    padding: 10px;
    margin-bottom: 20px;
    border: 1px solid #ddd;
    border-radius: 4px;
}

.actions button {
    padding: 10px 20px;
    margin: 5px;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    transition: background-color 0.3s;
}

.btn-check-in {
    background-color: #4CAF50;
    color: white;
}

.btn-check-out {
    background-color: #f44336;
    color: white;
}

.actions button:disabled {
    background-color: #cccccc;
    cursor: not-allowed;
}

.status-message {
    padding: 10px;
    margin-top: 20px;
    border-radius: 4px;
}

.success {
    background-color: #d4edda;
    color: #155724;
    border: 1px solid #c3e6cb;
}

.error {
    background-color: #f8d7da;
    color: #721c24;
    border: 1px solid #f5c6cb;
}

.history-section {
    margin-top: 25px;
    padding-top: 15px;
    border-top: 1px solid #eee;
    text-align: left;
}

.history-section h3 {
    text-align: center;
    margin-bottom: 15px;
}

.history-table {
    width: 100%;
    border-collapse: collapse;
    margin-top: 10px;
}

.history-table th,
.history-table td {
    border: 1px solid #ddd;
    padding: 8px;
    text-align: center;
}

.history-table th {
    background-color: #f2f2f2;
    font-weight: bold;
}

.history-table tr:nth-child(even) {
    background-color: #f9f9f9;
}

.history-table tr:hover {
    background-color: #f1f1f1;
}

.status-in {
    font-weight: bold;
    color: #155724;
}

.status-out {
    font-weight: bold;
    color: #721c24;
}

.btn-delete {
    background-color: #dc3545;
    color: white;
    border: none;
    border-radius: 4px;
    padding: 5px 8px;
    cursor: pointer;
    font-size: 12px;
    transition: background-color 0.3s, opacity 0.3s;
}

.btn-delete:hover:not(:disabled) {
    background-color: #c82333;
}

.btn-delete:disabled {
    background-color: #cccccc;
    cursor: not-allowed;
    opacity: 0.6;
}
</style>