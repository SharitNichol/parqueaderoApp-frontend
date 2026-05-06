<template>
  <ion-page>
    <ion-header>
      <ion-toolbar color="primary">
        <ion-title>Acceso Campus UCC</ion-title>
      </ion-toolbar>
    </ion-header>

    <ion-content class="ion-padding ion-text-center">
      <!-- ESTADO: Cargando datos -->
      <div v-if="isLoading" class="loading-state">
        <ion-spinner name="crescent" color="primary"></ion-spinner>
        <p>Verificando tu reserva en la Sede Av. 39...</p>
      </div>

      <!-- ESTADO: No hay reserva para hoy -->
      <div v-else-if="!reservaHoy" class="no-access">
        <div class="icon-circle">
          <ion-icon :icon="lockClosedOutline" color="danger"></ion-icon>
        </div>
        <h2>Acceso Restringido</h2>
        <p>No tienes una reserva activa para el día de hoy.</p>
        <p class="helper-text">Recuerda que debes reservar antes de ingresar al parqueadero.</p>
        <ion-button expand="block" router-link="/tabs/reservar" color="primary" class="ion-margin-top">
          Reservar Cupo Ahora
        </ion-button>
      </div>

      <!-- ESTADO: Reserva encontrada -> Mostrar QR -->
      <div v-else class="access-granted">
        <ion-card class="qr-card">
          <ion-card-header>
            <ion-badge color="success">RESERVA ACTIVA</ion-badge>
            <ion-card-title class="ion-margin-top">Pase de Ingreso</ion-card-title>
            <ion-card-subtitle>UCC - Sede Avenida 39</ion-card-subtitle>
          </ion-card-header>
          
          <ion-card-content>
            <div class="qr-wrapper">
              <canvas ref="qrCanvas"></canvas>
            </div>
            
            <div class="info-box">
              <p><strong>Estudiante:</strong> {{ usuario?.nombre || 'Usuario UCC' }}</p>
              <p><strong>Horario de Clase:</strong></p>
              <p class="time-highlight">{{ reservaHoy.hora_inicio_clase }} - {{ reservaHoy.hora_fin_clase }}</p>
            </div>
            
            <p class="footer-text">Presenta este código al guarda en la entrada.</p>
          </ion-card-content>
        </ion-card>
        
        <ion-button fill="clear" color="medium" @click="reintentarCarga">
          <ion-icon slot="start" :icon="refreshOutline"></ion-icon>
          Actualizar datos
        </ion-button>
      </div>
    </ion-content>
  </ion-page>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, watch } from 'vue';
import { useDashboard } from '@/composables/useDashboard';
import { lockClosedOutline, refreshOutline } from 'ionicons/icons';
import QRCode from 'qrcode';
import { 
  IonPage, IonHeader, IonToolbar, IonTitle, IonContent, IonIcon, 
  IonButton, IonCard, IonCardHeader, IonCardTitle, IonCardSubtitle, 
  IonCardContent, IonBadge, IonSpinner 
} from '@ionic/vue';

const { reservasActivas, cargarDatos, usuario, isLoading } = useDashboard();
const qrCanvas = ref<HTMLCanvasElement | null>(null);

// Lógica de detección de reserva para HOY (formato YYYY-MM-DD)
const reservaHoy = computed(() => {
  const hoy = new Date().toLocaleDateString('en-CA'); 
  return reservasActivas.value.find(r => {
    const fechaReserva = new Date(r.fecha_entrada).toLocaleDateString('en-CA');
    return fechaReserva === hoy && r.estado === 'activa';
  });
});

const generarQR = async () => {
  if (reservaHoy.value && qrCanvas.value) {
    const payload = JSON.stringify({
      rid: reservaHoy.value.id,
      uid: usuario.value?.id,
      sede: "UCC-AV39",
      horario: `${reservaHoy.value.hora_inicio_clase}-${reservaHoy.value.hora_fin_clase}`,
      ts: new Date().getTime()
    });

    try {
      await QRCode.toCanvas(qrCanvas.value, payload, {
        width: 240,
        margin: 1,
        color: { dark: '#003366', light: '#ffffff' } // Colores UCC
      });
    } catch (err) {
      console.error("Error al generar QR:", err);
    }
  }
};

const reintentarCarga = async () => {
  await cargarDatos();
  setTimeout(generarQR, 500);
};

onMounted(async () => {
  await cargarDatos();
  setTimeout(generarQR, 500);
});

// Observar si la reserva cambia para regenerar el QR
watch(reservaHoy, () => {
  setTimeout(generarQR, 500);
});
</script>

<style scoped>
.loading-state { margin-top: 100px; }
.qr-wrapper {
  display: flex;
  justify-content: center;
  background: white;
  padding: 10px;
  border-radius: 12px;
  margin: 10px 0;
}
.qr-card { border-radius: 20px; overflow: hidden; }
.no-access { margin-top: 60px; padding: 20px; }
.icon-circle {
  width: 120px;
  height: 120px;
  background: #fff5f5;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  margin: 0 auto 20px;
}
.icon-circle ion-icon { font-size: 60px; }
.info-box {
  background: #f8f9fa;
  padding: 15px;
  border-radius: 12px;
  margin: 15px 0;
  text-align: center;
  border: 1px solid #eee;
}
.time-highlight {
  font-size: 1.4rem;
  font-weight: bold;
  color: var(--ion-color-primary);
  margin: 5px 0;
}
.helper-text { color: #666; font-size: 0.9rem; }
.footer-text { font-size: 0.8rem; color: #999; margin-top: 10px; }
</style>
