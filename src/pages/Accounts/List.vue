<template>
  <div class="accounts-list">
    <ListItem v-for="(account, index) in accounts" :key="index" :id="account.id" :name="account.name" :img="account.img"
              :active="account.active" :phone_number="account.phone_number" @make-main="makeMain"></ListItem>
  </div>
</template>

<script>
import ListItem from "@/components/Accounts/ListItem.vue";
import axios from 'axios'; // make sure to import axios

export default {
  components: {
    ListItem
  },
  name: "List",
  async mounted() {
    this.$root.$on('update_list', () => {
      this.update_list()
    })
    await this.update_list(); // call update_list on mount to fetch initial data
  },
  methods: {

    async update_list() {

      try {
        const response = await axios.get(`${process.env.VUE_APP_BASE_API_URL}/accounts`, {
          headers: {
            Authorization: localStorage.getItem("auth_token")
          }
        });
        console.log(response.data)
        const data = response.data;

        // Get the main account
        const mainResponse = await axios.get(`${process.env.VUE_APP_BASE_API_URL}/main_account`, {
          headers: {
            Authorization: localStorage.getItem("auth_token")
          }
        });
        console.log(mainResponse.data)
        let mainUserId = 0
        if (mainResponse.data !== null) {
          mainUserId = mainResponse.data.__data__.id;
        }


        const accounts = [];
        for (const item of data) {
          const account = {
            id: item.__data__.id,
            name: item.__data__.name,
            phone_number: item.__data__.phone_number,
            active: item.__data__.id === mainUserId,
          };


          accounts.push(account);
        }

        this.accounts = accounts;
      } catch (error) {
        console.error('Failed to update list:', error);
      }
      this.stop_loading();
    },
    stop_loading: function () {
      this.$root.$emit('stop_loading')
    },
    async makeMain(id) {
      try {
        const response = await axios.post(`${process.env.VUE_APP_BASE_API_URL}/main_account`, id, {
          headers: {
            Authorization: localStorage.getItem("auth_token")
          }
        });
        console.log(response.data.message);
        await this.update_list();
      } catch (error) {
        console.error('Failed to make account main:', error);
      }
    },
     async checkSessionValidity(sessionId) {
        try {
            const response = await axios.get(`${process.env.VUE_APP_BASE_API_URL}/check_invalid?session_id=${sessionId}`, {
                headers: {
                    Authorization: localStorage.getItem("auth_token")
                }
            });
            return response.data.valid;  // Assuming the response contains a 'valid' field
        } catch (error) {
            console.error(`Error checking session validity: ${error}`);
            return false;
        }
    },
  },
  data() {
    return {
      accounts: []
    }
  },
}
</script>

<style scoped></style>
