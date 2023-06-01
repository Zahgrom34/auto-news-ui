<template>
  <div v-if="isVisible" class="account list-item">
    <div class="account list-item">
      <div class="account-icon">
        <img :src="img" alt="AccountName">
      </div>
      <div class="account-details">
        <div class="account-body">
          <h4 class="font-medium p-0 m-0 pb-2">{{ name }}</h4>
        </div>
        <span :style="{ color: active ? 'green' : 'red' }">
            {{ active ? 'Active' : 'Inactive' }}
        </span>
      </div>
      <span class="text-muted">{{ formattedPhoneNumber }}</span>
      <div @click="deleteAccount" class="tim-icons icon-simple-remove"></div>
      <button class="absurd-button" @click="makeMain">Make Main</button>
      <button v-if="sessionInvalid" @click="refreshSession">Refresh Session</button>
      <div v-if="code_retrieval_state">
        <label>Enter the code</label>
        <input v-model="code_received" type="number" class="form-control" placeholder="Code from telegram">
        <button @click="submitCode">Submit Code</button>
      </div>
    </div>
  </div>
</template>


<script>
import {parsePhoneNumberFromString} from 'libphonenumber-js';
import axios from "axios";

export default {
  name: "list-item",
  props: {
    id: Number,
    name: {
      type: String,
      required: true,
    },
    img: String,
    active: Boolean,
    phone_number: {
      type: String,
      required: true
    }
  },
  computed: {
    formattedPhoneNumber() {
      const phoneNumber = parsePhoneNumberFromString(this.phone_number);
      if (phoneNumber) {
        const nationalNumber = phoneNumber.nationalNumber;
        return '+' + phoneNumber.countryCallingCode + '(' + nationalNumber.slice(0, 3) + ') ' + nationalNumber.slice(3, 6) + '-' + nationalNumber.slice(6, 8) + '-' + nationalNumber.slice(8, 10);
      }
      return this.phone_number;
    },
  },
  methods: {
    async deleteAccount() {
      try {
        const response = await axios.delete(`${process.env.VUE_APP_BASE_API_URL}/delete_session/${this.id}`, {
          headers: {
            Authorization: localStorage.getItem("auth_token")
          }
        });
        console.log(response.data.message);
        this.isVisible = false
      } catch (error) {
        console.error('Failed to delete account:', error);
        this.isVisible = false
      }
    },
    makeMain() {
      this.$emit('make-main', this.id);
    },
    async refreshSession() {
      // Delete the session
      try {
        // Request the actual telegram code
        const sessionIdResponse = await axios.post(`${process.env.VUE_APP_BASE_API_URL}/create_session?session_id=${this.id}`, null, {
          headers: {
            Authorization: localStorage.getItem("auth_token")
          }
        });
        this.hash_code = sessionIdResponse.data.hash_code;

        // Request the code received as input from the user
        this.code_retrieval_state = true;

      } catch (error) {
        console.error('Failed to create new session:', error);
      }
    },
    async submitCode() {
      try {
        const url = `${process.env.VUE_APP_BASE_API_URL}/auth_session/?session_id=${this.session_name}&hash_code=${this.hash_code}&code=${this.code_received}`;
        const response = await axios.post(url, {
          headers: {
            Authorization: localStorage.getItem("auth_token")
          }
        });
        if (response.data.code === 200) {
          await this.notifyVue('top', 'right', `Logged in with ${this.name}`, "success");
          this.code_retrieval_state = false;
        } else {
          console.error(`Unexpected response code: ${response.data.code}`);
          await this.notifyVue('top', 'right', `${response.data.detail}`)
        }
      } catch (error) {
        console.error(`Error sending request: ${error}`);
        await this.notifyVue('top', 'right', `${error}`)
      }
      this.code_received = null
    },
    async checkSessionValidity() {
      this.sessionInvalid = !(await this.$parent.checkSessionValidity(this.id));
    },
  },
  data() {
    return {
      isVisible: true,
      code_retrieval_state: false,
      code_received: null,
      hash_code: null,
      session_name: null,
      sessionInvalid: false
    }
  },

  async mounted() {
    await this.checkSessionValidity();
  },

}
</script>

<style lang="scss" scoped>
.account-details {
  display: flex;
  flex-direction: column;
}

.list-item {
  position: relative;
}
</style>
