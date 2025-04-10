<template>
  <ion-page>
    <ion-header>
      <ion-toolbar color="primary">
        <ion-title><ion-icon :icon="qrCode" slot="start" /> Escáner QR</ion-title>
      </ion-toolbar>
    </ion-header>

    <ion-content class="ion-padding">
      <ion-button expand="block" @click="toggleScan" :disabled="scanning">
        <ion-icon :icon="scanning ? stop : camera" slot="start" />
        {{ scanning ? 'Detener Escaneo' : 'Escanear QR' }}
      </ion-button>

      <div v-if="scanning" class="scanner-container">
        <video ref="videoElement" playsinline></video>
        <div class="scan-overlay" />
      </div>

      <ion-card v-if="currentScan" class="result-card">
        <ion-card-header>
          <ion-card-title>Resultado del Escaneo</ion-card-title>
          <ion-card-subtitle>{{ getType(currentScan.content) }}</ion-card-subtitle>
        </ion-card-header>
        <ion-card-content>
          <div v-if="isText(currentScan.content)" class="text-content">{{ currentScan.content }}</div>
          <ion-button v-else expand="block" @click="handleSpecial(currentScan.content)">
            {{ isUrl(currentScan.content) ? 'Abrir Enlace' : 'Enviar Email' }}
          </ion-button>
        </ion-card-content>
      </ion-card>

      <ion-list lines="full" v-if="history.length">
        <ion-list-header>
          <ion-label>Historial de Escaneos</ion-label>
          <ion-button fill="clear" @click="confirmClear"><ion-icon :icon="trash" slot="icon-only" /></ion-button>
        </ion-list-header>

        <ion-item-sliding v-for="(scan, i) in history" :key="i">
          <ion-item @click="currentScan = scan">
            <ion-icon :icon="getIcon(scan.content)" :color="getColor(scan.content)" slot="start" />
            <ion-label>
              <h3>{{ formatContent(scan.content) }}</h3>
              <p>{{ formatDate(scan.timestamp) }}</p>
            </ion-label>
          </ion-item>
          <ion-item-options side="end">
            <ion-item-option color="danger" @click="deleteScan(i)">
              <ion-icon :icon="trash" />
            </ion-item-option>
          </ion-item-options>
        </ion-item-sliding>
      </ion-list>

      <div v-else-if="!currentScan" class="empty-state">
        <ion-icon :icon="qrCode" />
        <p>Escanea un código QR para comenzar</p>
      </div>
    </ion-content>
  </ion-page>
</template>

<script>
import {
  IonPage, IonHeader, IonToolbar, IonTitle, IonContent,
  IonButton, IonIcon, IonCard, IonCardHeader, IonCardTitle,
  IonCardSubtitle, IonCardContent, IonList, IonListHeader,
  IonItem, IonLabel, IonItemSliding, IonItemOptions, IonItemOption,
  alertController, toastController
} from '@ionic/vue';
import { qrCode, camera, stop, trash, link, mail, documentText } from 'ionicons/icons';
import { Browser } from '@capacitor/browser';

export default {
  components: {
    IonPage, IonHeader, IonToolbar, IonTitle, IonContent,
    IonButton, IonIcon, IonCard, IonCardHeader, IonCardTitle,
    IonCardSubtitle, IonCardContent, IonList, IonListHeader,
    IonItem, IonLabel, IonItemSliding, IonItemOptions, IonItemOption
  },
  data: () => ({
    scanning: false,
    currentScan: null,
    history: [],
    codeReader: null,
    qrCode, camera, stop, trash, link, mail, documentText
  }),
  async mounted() {
    const data = localStorage.getItem('qrHistory');
    if (data) this.history = JSON.parse(data);
  },
  methods: {
    toggleScan() {
      this.scanning ? this.stopScanner() : this.startScanner();
    },
    async startScanner() {
      this.scanning = true;
      try {
        const { BrowserQRCodeReader } = await import('@zxing/browser');
        this.codeReader = new BrowserQRCodeReader();
        this.codeReader.decodeFromVideoDevice(null, this.$refs.videoElement, (res, err) => {
          if (res) {
            this.processResult(res.text);
            this.codeReader.reset();
            this.scanning = false;
          }
          if (err) console.log(err);
        });
      } catch (e) {
        console.error(e);
        this.showToast('Error al acceder a la cámara');
        this.scanning = false;
      }
    },
    stopScanner() {
      this.codeReader?.reset();
      this.scanning = false;
    },
    processResult(content) {
      const scan = { content, timestamp: new Date().toISOString() };
      this.currentScan = scan;
      this.history.unshift(scan);
      localStorage.setItem('qrHistory', JSON.stringify(this.history));
      this.isUrl(content) ? this.openLink(content) : this.isEmail(content) && this.sendEmail(content);
    },
    deleteScan(i) {
      this.history.splice(i, 1);
      localStorage.setItem('qrHistory', JSON.stringify(this.history));
    },
    async confirmClear() {
      const alert = await alertController.create({
        header: 'Confirmar',
        message: '¿Borrar todo el historial?',
        buttons: [{ text: 'Cancelar', role: 'cancel' }, { text: 'Borrar', handler: this.clearHistory }]
      });
      await alert.present();
    },
    clearHistory() {
      this.history = [];
      localStorage.removeItem('qrHistory');
    },
    isUrl: c => { try { return !!new URL(c); } catch { return false; } },
    isEmail: c => /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(c),
    isText(c) { return !this.isUrl(c) && !this.isEmail(c); },
    getType(c) { return this.isUrl(c) ? 'Enlace web' : this.isEmail(c) ? 'Dirección de email' : 'Texto'; },
    getIcon(c) { return this.isUrl(c) ? this.link : this.isEmail(c) ? this.mail : this.documentText; },
    getColor(c) { return this.isUrl(c) ? 'primary' : this.isEmail(c) ? 'secondary' : 'medium'; },
    formatContent: c => c.length > 30 ? c.slice(0, 30) + '...' : c,
    formatDate: ts => new Date(ts).toLocaleString(),
    async openLink(url) {
      try { await Browser.open({ url }); } catch { window.open(url, '_blank'); }
    },
    sendEmail(email) {
      window.open(`mailto:${email}`);
    },
    handleSpecial(content) {
      this.isUrl(content) ? this.openLink(content) : this.sendEmail(content);
    },
    async showToast(msg) {
      const toast = await toastController.create({ message: msg, duration: 2000, position: 'top' });
      await toast.present();
    }
  },
  beforeUnmount() {
    this.stopScanner();
  }
};
</script>

<style scoped>
.scanner-container {
  position: relative;
  width: 100%;
  max-width: 400px;
  margin: 20px auto;
  border: 2px solid var(--ion-color-primary);
  border-radius: 8px;
  overflow: hidden;
}
.scanner-container video { width: 100%; display: block; }
.scan-overlay {
  position: absolute; top: 50%; left: 50%;
  transform: translate(-50%, -50%);
  width: 70%; height: 70%;
  border: 3px solid #00ff00;
  box-shadow: 0 0 0 100vmax rgba(0,0,0,0.5);
}
.result-card { margin-top: 20px; }
.text-content {
  padding: 10px;
  background: var(--ion-color-light);
  border-radius: 5px;
  word-break: break-word;
}
.empty-state {
  text-align: center;
  margin-top: 50px;
  color: var(--ion-color-medium);
}
.empty-state ion-icon {
  font-size: 64px;
  margin-bottom: 16px;
}
</style>
