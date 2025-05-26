<template>
  <div class="container">
    <div class="card">
      <!-- Logo -->
      <img :src="logo" alt="Logo" class="logo" />

      <!-- Texto principal -->
      <h2 class="title">MODIFICAR FECHA ENTREGA PEDIDO DE CLIENTE</h2>

      <!-- Cuadro interno -->

      <form @submit.prevent="buscarPedido">
        <div class="input-section">
          <label class="input-label">Ingrese número de pedido</label>
          <input
            type="text"
            v-model="numeroPedido"
            class="input-box"
            placeholder="Número de pedido"
          />
          <p v-if="error" class="error-text">{{ error }}</p>
        </div>
  
        <!-- Botones -->
        <div class="buttons">
          <button
            @click="buscarPedido"
            class="btn btn-buscar"
            :disabled="cargando"
          >
            {{ cargando ? 'Buscando...' : 'Buscar' }}
          </button>
          <button @click.prevent="limpiar" class="btn btn-limpiar">Limpiar</button>
        </div>
      </form>
    </div>

    <!-- Resultado de búsqueda -->
    <div v-if="list_detalles.length > 0" class="resultado">
      <div class="resumen">
        <p><strong>Numero:</strong> {{ numeroPedido }}</p>
        <p><strong>Cliente:</strong> {{ tercero }}</p>
        <p><strong>Fecha:</strong> {{ fecha }}</p>
      </div>
      <hr />

      <div class="row-indicadores">
        <div class="right">
          <input type="checkbox" id="checkIndicadores" v-model="checkIndicadores" />
          <label for="checkIndicadores">Cambiar en indicadores</label>
          <input type="date" v-model="fechaIndicador" class="input-fecha-indicador" />
          <button class="btn btn-guardar" @click="guardarIndicadores">Guardar</button>
        </div>
      </div>

      <hr />

      <table class="tabla">
        <thead>
          <tr>
            <th>Código</th>
            <th>Descripción</th>
            <th>Cantidad</th>
            <th>Valor Unitario</th>
            <th>Fecha Entrega</th>
            <th>Nueva Fecha</th>
            <th>Actualizar</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="(item, index) in list_detalles" :key="index">
            <td>{{ item.codigo }}</td>
            <td>{{ item.nombre_producto }}</td>
            <td>{{ item.cantidad }}</td>
            <td>{{ item.valor_unitario }}</td>
            <td>{{ item.fecha_entrega }}</td>
            <td>
              <input type="date" v-model="item.nueva_fecha" />
            </td>
            <td>
              <button @click="actualizarFila(item)" class="btn-icono">🔄</button>
            </td>
          </tr>
        </tbody>
      </table>
    </div>
  </div>

  <!-- Overlay de carga -->
  <div v-if="loading" class="loading-overlay">
      <div class="spinner-border text-light" role="status">
          <span class="visually-hidden"></span>
      </div>
      <p class="mt-2 text-light">{{ loading_msg }}</p>
  </div>

</template>

<script setup>
import apiUrl from "../../config.js";
import { ref, watch } from 'vue'
import axios from 'axios'
import logo from '@/assets/logo.png'

const msg = ref('');
const nit = ref('');
const tercero = ref('');
const fecha = ref(null);
const numeroPedido = ref('');
const error = ref('');
const cargando = ref(false);
const pedido = ref(null);
const list_detalles = ref([]);
const checkIndicadores = ref(false);
const fechaIndicador = ref(null);

const loading = ref(false);
const loading_msg = ref('');

const buscarPedido = async () => {
  error.value = ''
  pedido.value = null

  if (!numeroPedido.value.trim()) {
    error.value = 'Por favor ingrese un número de pedido.'
    return
  }

  cargando.value = true

  try {
    const response = await axios.post(
        `${apiUrl}/consultar_pedido`,
        {
            numero_pedido: numeroPedido.value
        },
        {
            headers: {
                Accept: "application/json"
            }
        }
    );
    if (response.status === 200) {
      if (response.data.data.length === 0) {
        list_detalles.value = []
        alert('No se encontraron detalles para el pedido ingresado.')
        return
      }
      nit.value = response.data.data.nit;
      tercero.value = response.data.data.tercero;
      fecha.value = response.data.data.fecha;
      list_detalles.value = response.data.data.detalles;

      // Agrega campos editables al detalle
      list_detalles.value = list_detalles.value.map(item => ({
        ...item,
        // Para usar en el input
        nueva_fecha: convertirMDY_a_YMD(item.nueva_fecha),
      }));

    }

  } catch (error) {
    console.error('Error al cargar los datos:', error);
  } finally {
    cargando.value = false
  }
}

function convertirMDY_a_YMD(fechaStr) {
  if (!fechaStr || !fechaStr.includes('-')) return '';
  const [mes, dia, anio] = fechaStr.split('-');
  if (!mes || !dia || !anio) return '';
  return `${anio}-${mes.padStart(2, '0')}-${dia.padStart(2, '0')}`;
}

const limpiar = () => {
  numeroPedido.value = '';
  error.value = '';
  pedido.value = null;
  list_detalles.value = [];
  checkIndicadores.value = false;
  fechaIndicador.value = null;
}

const actualizarFila = async (item) => {

  const confirmado = window.confirm('¿Está seguro que desea actualizar este item?');
  if (!confirmado) return;

  try {
    const response = await axios.post(
      `${apiUrl}/actualizar_fecha_individual`,
      {
        numero_pedido: numeroPedido.value,
        checkIndicadores: checkIndicadores.value,
        item_detalle: item
      },
      {
        headers: {
          Accept: "application/json"
        }
      }
    );
    if (response.status === 200) {
      msg.value = response.data.message;
      await buscarPedido();
      alert(msg.value);
    }
  } catch (error) {
    alert('Error al guardar');
    console.error(error);
  }
}

const guardarIndicadores = async () => {
  if (!fechaIndicador.value) {
    alert('Por favor seleccione una fecha para el indicador.');
    return;
  }
  const confirmado = window.confirm('¿Está seguro que desea realizar el guardado masivo?');
  if (!confirmado) return;

  try {
    loading.value = true;
    loading_msg.value = 'Guardando datos, por favor espere...';
    const response = await axios.post(
      `${apiUrl}/actualizar_masivo`,
      {
        numero_pedido: numeroPedido.value,
        checkIndicadores: checkIndicadores.value,
        detalles: list_detalles.value
      },
      {
        headers: {
          Accept: "application/json"
        }
      }
    );
    if (response.status === 200) {
      msg.value = response.data.message;
      await buscarPedido();
      alert(msg.value);
    }
  } catch (error) {
    alert('Error al guardar');
    console.error(error);
  } finally {
    loading.value = false;
    checkIndicadores.value = false;
    fechaIndicador.value = null;
  }
}

watch(fechaIndicador, (nuevaFecha) => {
  if (nuevaFecha) {
    list_detalles.value = list_detalles.value.map(item => ({
      ...item,
      nueva_fecha: nuevaFecha
    }))
  }
})

</script>

<style scoped>

.container {
  min-width: 100vw;
  width: 100%;
  padding: 20px;
  background-color: #f4f4f4;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  border-radius: 10px;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.1);
}

.card {
  width: 320px;
  background-color: #fff;
  border: 1px solid #ddd;
  border-radius: 10px;
  padding: 16px;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.1);
}

.logo {
  display: block;
  margin: 0 auto 10px;
  width: 150px;
  height: auto;
}

.title {
  font-size: 14px;
  text-align: center;
  font-weight: bold;
  margin-bottom: 16px;
  color: #333;
}

.input-section {
  background-color: #f0f0f0;
  padding: 12px;
  border-radius: 6px;
  margin-bottom: 16px;
}

.input-label {
  display: block;
  margin-bottom: 8px;
  font-size: 13px;
  color: #555;
}

.input-box {
  width: 100%;
  padding: 8px;
  font-size: 14px;
  border: 1px solid #bbb;
  border-radius: 4px;
}

.buttons {
  display: flex;
  justify-content: space-between;
}

.btn {
  padding: 8px 16px;
  font-size: 14px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.btn-buscar {
  background-color: #ffd95e;
  color: black;
}

.btn-buscar:hover {
  background-color: #ffd343;
  color: white;
}

.btn-limpiar {
  background-color: #940404;
  color: white;
}

.btn-limpiar:hover {
  background-color: #f84f4f;
  color: white;
}

.error-text {
  color: red;
  font-size: 13px;
  margin-top: 6px;
}

.resultado {
  margin-top: 20px;
  background-color: #fff;
  padding: 16px;
  border-radius: 10px;
  border: 1px solid #ddd;
}

.resumen {
  margin-bottom: 12px;
}

.resumen p {
  margin: 4px 0;
  font-size: 13px;
}

.row-indicadores {
  display: flex;
  justify-content: flex-end;
  align-items: center;
  margin-bottom: 12px;
}

.right input[type="checkbox"] {
  margin-right: 8px;
}

.right label{
  margin-right: 12px;
}

.right {
  display: flex;
  align-items: center;
}

.input-fecha-indicador {
  padding: 4px;
  font-size: 12px;
  border: 1px solid #bbb;
  border-radius: 4px;
  margin-right: 8px;
}

.btn-guardar {
  background-color: #5cb85c;
  color: white;
  padding: 6px 12px;
  font-size: 12px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.btn-guardar:hover {
  background-color: #4cae4c;
}

.tabla {
  width: 100%;
  border-collapse: collapse;
  font-size: 12px; /* más pequeño */
}

.tabla th,
.tabla td {
  border: 1px solid #e5e7eb;
  padding: 3px 4px; /* menos padding */
  text-align: center;
  height: 28px; /* menor alto */
  line-height: 1.1;
}

th {
    background-color: #fff7dd;
}

.tabla input[type="date"] {
  width: 100%;
  font-size: 12px; /* más pequeño */
  padding: 2px 4px; /* menos padding */
  height: 24px; /* menor alto */
}

.btn-icono {
  background: none;
  border: none;
  padding: 4px 8px;
  font-size: 18px;
  cursor: pointer;
  transition: color 0.2s, transform 0.2s;
  color: #ffd95e;
  outline: none;
}

.btn-icono:hover {
  color: #ffb300;
  transform: scale(1.2) rotate(-10deg);
  background: none;
  box-shadow: none;
}

.tabla tr:hover {
  background-color: #fffbe6;
  box-shadow: 0 2px 8px 0 rgba(255, 217, 94, 0.10);
  transition: background 0.2s, box-shadow 0.2s;
}

.loading-overlay {
    position: fixed;
    top: 0;
    left: 0;
    width: 100vw;
    height: 100vh;
    background: rgba(0, 0, 0, 0.7); /* Fondo oscuro */
    display: flex;
    justify-content: center;
    align-items: center;
    z-index: 9999; /* Asegura que esté sobre todo */
}

</style>