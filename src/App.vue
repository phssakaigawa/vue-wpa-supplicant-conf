<template>
  <div>
    <form @submit="addItem" autocomplete="off">
      SSID: <input type="text" name="newSSID" ref="ssid">
      Passphrase: <input type="password" name="newPassphrase">
      <button type="submit">追加</button>
    </form>
    <ul>
      <li v-for="item in items">
        SSID: {{ item.id }},
        Security: <span :class="{ 'sec-weak' : item.psk.length === 0 }">{{ (item.psk.length > 0) ? 'WPA2' : 'None' }}</span>
        <button @click="deleteItem(item)" class="button-small button-red">削除</button>
      </li>
    </ul>
    <a :href="urlWPA" @click="downloadWPA" download="wpa_supplicant.conf">
      <button>wpa_supplicant.conf作成</button>
    </a>
    <a :href="urlSSH" @click="downloadSSH" download="ssh.txt">
      <button>ssh.txt作成</button>
    </a>
  </div>
</template>

<script>
import PBKDF2 from 'crypto-js/pbkdf2';

export default {
  data() {
    return {
      items: [],
      urlWPA: '#',
      urlSSH: '#',
    };
  },
  beforeMount() {
    this.updateUrlWPA();
    if (!window.navigator.msSaveBlob) {
      this.urlSSH = window.URL.createObjectURL(this.makeBlob(''));
    }
  },
  mounted() {
    this.$refs.ssid.focus();
  },
  methods: {
    addItem(e) {
      e.preventDefault();

      const newSSID = e.target.newSSID.value;
      const newPassphrase = e.target.newPassphrase.value;

      if (newSSID.length === 0) return;

      this.items.push({
        id: newSSID,
        psk: newPassphrase.length !== 0 ? PBKDF2(newPassphrase, newSSID, { keySize: 8, iterations: 4096 }).toString() : '',
      });
      this.updateUrlWPA();

      e.target.newSSID.value = '';
      e.target.newPassphrase.value = '';
      e.target.newSSID.focus();
    },
    deleteItem(item) {
      const index = this.items.indexOf(item);
      this.items.splice(index, 1);
      this.updateUrlWPA();
    },
    makeConfig() {
      let config = 'country=JP\n';
      config += 'ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev\n';
      config += 'update_config=1\n';

      config += this.items.map((item) => {
        let network = 'network={\n';
        network += `    ssid="${item.id}"\n`;
        if (item.psk.length > 0) {
          network += `    psk=${item.psk}\n`;
        } else {
          network += '    key_mgmt=NONE\n';
        }
        network += '}';
        return network;
      }).join('\n');

      return config;
    },
    updateUrlWPA() {
      if (!window.navigator.msSaveBlob) {
        const content = this.makeConfig();
        this.urlWPA = window.URL.createObjectURL(this.makeBlob(content));
      }
    },
    downloadWPA() {
      if (window.navigator.msSaveBlob) {
        const fileName = 'wpa_supplicant.conf';
        const content = this.makeConfig();
        window.navigator.msSaveBlob(this.makeBlob(content), fileName);
      }
    },
    downloadSSH() {
      if (window.navigator.msSaveBlob) {
        const fileName = 'ssh.txt';
        window.navigator.msSaveBlob(this.makeBlob(''), fileName);
      }
    },
    makeBlob(content) {
      return new Blob([content], { type: 'text/plain' });
    },
  },
};
</script>
