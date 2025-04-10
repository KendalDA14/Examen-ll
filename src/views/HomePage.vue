<template>
  <ion-page>
    <ion-header>
      <ion-toolbar color="primary">
        <ion-title>
          <ion-icon :icon="qrCode" slot="start"></ion-icon>
          Escáner QR
        </ion-title>
      </ion-toolbar>
    </ion-header>

    <ion-content class="ion-padding">
      <ion-button expand="block" @click="scanQR" :disabled="scanning">
        <ion-icon :icon="scanning ? stop : camera" slot="start"></ion-icon>
        {{ scanning ? 'Detener Escaneo' : 'Escanear QR' }}
      </ion-button>
      <div class="scanner-container" v-if="scanning">
        <video ref="videoElement" playsinline></video>
        <div class="scan-overlay"></div>
      </div>

      <ion-card v-if="currentScan" class="result-card">
        <ion-card-header>
          <ion-card-title>Resultado del Escaneo</ion-card-title>
          <ion-card-subtitle>{{ getType(currentScan.content) }}</ion-card-subtitle>
        </ion-card-header>
        <ion-card-content>
          <div v-if="isText(currentScan.content)" class="text-content">
            {{ currentScan.content }}
          </div>
          <ion-button v-if="isUrl(currentScan.content)" expand="block" @click="openLink(currentScan.content)">
            Abrir Enlace
          </ion-button>
          <ion-button v-if="isEmail(currentScan.content)" expand="block" @click="sendEmail(currentScan.content)">
            Enviar Email
          </ion-button>
        </ion-card-content>
      </ion-card>

      <ion-list lines="full" v-if="history.length > 0">
        <ion-list-header>
          <ion-label>Historial de Escaneos</ion-label>
          <ion-button fill="clear" @click="confirmClear">
            <ion-icon :icon="trash" slot="icon-only"></ion-icon>
          </ion-button>
        </ion-list-header>

        <ion-item-sliding v-for="(scan, index) in history" :key="index">
          <ion-item @click="showScanDetails(scan)">
            <ion-icon :icon="getIcon(scan.content)" slot="start" :color="getColor(scan.content)"></ion-icon>
            <ion-label>
              <h3>{{ formatContent(scan.content) }}</h3>
              <p>{{ formatDate(scan.timestamp) }}</p>
            </ion-label>
          </ion-item>
          <ion-item-options side="end">
            <ion-item-option color="danger" @click="deleteScan(index)">
              <ion-icon :icon="trash"></ion-icon>
            </ion-item-option>
          </ion-item-options>
        </ion-item-sliding>
      </ion-list>

      <div class="empty-state" v-if="!currentScan && history.length === 0">
        <ion-icon :icon="qrCode"></ion-icon>
        <p>Escanea un código QR para comenzar</p>
      </div>
    </ion-content>
  </ion-page>
</template>

<script>
import {
  IonPage, IonHeader, IonToolbar, IonTitle, IonContent,
  IonButton, IonIcon, IonCard, IonCardHeader,
  IonCardTitle, IonCardSubtitle, IonCardContent,
  IonList, IonListHeader, IonItem, IonLabel,
  IonItemSliding, IonItemOptions, IonItemOption,
  alertController, toastController
} from '@ionic/vue';
import {
  qrCode, camera, stop, trash,
  link, mail, documentText, time
} from 'ionicons/icons';
import { Browser } from '@capacitor/browser';

export default {
  components: {
    IonPage, IonHeader, IonToolbar, IonTitle, IonContent,
    IonButton, IonIcon, IonCard, IonCardHeader,
    IonCardTitle, IonCardSubtitle, IonCardContent,
    IonList, IonListHeader, IonItem, IonLabel,
    IonItemSliding, IonItemOptions, IonItemOption
  },
  data() {
    return {
      scanning: false,
      currentScan: null,
      history: [],
      codeReader: null,
      qrCode, camera, stop, trash,
      link, mail, documentText, time
    };
  },
  async mounted() {
    await this.loadHistory();
  },
  methods: {
    async scanQR() {
      if (this.scanning) {
        this.stopScanner();
        return;
      }

      this.scanning = true;
      
      try {
        const { BrowserQRCodeReader } = await import('@zxing/browser');
        this.codeReader = new BrowserQRCodeReader();
        
        const videoElement = this.$refs.videoElement;
        const result = await this.codeReader.decodeFromVideoDevice(
          null, 
          videoElement, 
          (result, err) => {
            if (result) {
              this.processResult(result.text);
              this.codeReader.reset();
              this.scanning = false;
            }
            if (err) console.log(err);
          }
        );
      } catch (error) {
        console.error(error);
        this.showToast('Error al acceder a la cámara');
        this.scanning = false;
      }
    },

    stopScanner() {
      if (this.codeReader) {
        this.codeReader.reset();
      }
      this.scanning = false;
    },

    processResult(content) {
      this.currentScan = {
        content,
        timestamp: new Date().toISOString()
      };
      
      this.addToHistory(this.currentScan);
      
      // Ejecutar acción automática
      if (this.isUrl(content)) {
        this.openLink(content);
      } else if (this.isEmail(content)) {
        this.sendEmail(content);
      }
    },

    addToHistory(scan) {
      this.history.unshift(scan);
      localStorage.setItem('qrHistory', JSON.stringify(this.history));
    },

    async loadHistory() {
      const history = localStorage.getItem('qrHistory');
      if (history) {
        this.history = JSON.parse(history);
      }
    },

    showScanDetails(scan) {
      this.currentScan = scan;
    },

    deleteScan(index) {
      this.history.splice(index, 1);
      this.saveHistory();
    },

    async confirmClear() {
      const alert = await alertController.create({
        header: 'Confirmar',
        message: '¿Borrar todo el historial?',
        buttons: [
          { text: 'Cancelar', role: 'cancel' },
          { text: 'Borrar', handler: () => this.clearHistory() }
        ]
      });
      await alert.present();
    },

    clearHistory() {
      this.history = [];
      localStorage.removeItem('qrHistory');
    },

    saveHistory() {
      localStorage.setItem('qrHistory', JSON.stringify(this.history));
    },

    isText(content) {
      return !this.isUrl(content) && !this.isEmail(content);
    },

    isUrl(content) {
      try {
        new URL(content);
        return true;
      } catch {
        return false;
      }
    },

    isEmail(content) {
      return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(content);
    },

    getType(content) {
      if (this.isUrl(content)) return 'Enlace web';
      if (this.isEmail(content)) return 'Dirección de email';
      return 'Texto';
    },

    getIcon(content) {
      if (this.isUrl(content)) return link;
      if (this.isEmail(content)) return mail;
      return documentText;
    },

    getColor(content) {
      if (this.isUrl(content)) return 'primary';
      if (this.isEmail(content)) return 'secondary';
      return 'medium';
    },

    formatContent(content) {
      if (content.length > 30) {
        return content.substring(0, 30) + '...';
      }
      return content;
    },

    formatDate(timestamp) {
      return new Date(timestamp).toLocaleString();
    },

    async openLink(url) {
      try {
        await Browser.open({ url });
      } catch {
        window.open(url, '_blank');
      }
    },

    sendEmail(email) {
      window.open(`mailto:${email}`);
    },

    async showToast(message) {
      const toast = await toastController.create({
        message,
        duration: 2000,
        position: 'top'
      });
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

.scanner-container video {
  width: 100%;
  display: block;
}

.scan-overlay {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 70%;
  height: 70%;
  border: 3px solid #00ff00;
  box-shadow: 0 0 0 100vmax rgba(0, 0, 0, 0.5);
}

.result-card {
  margin-top: 20px;
}

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