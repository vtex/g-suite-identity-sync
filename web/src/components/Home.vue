<template>
  <div>
    <form novalidate class="md-layout md-gutter" @submit.prevent="validateForm">
      <md-card class="md-layout-item md-size-40 md-small-size-100">
        <md-progress-bar md-mode="indeterminate" v-if="showProgress"/>
        <md-card-header>
          <div class="md-title">Welcome to VTEX Wi-Fi</div>
        </md-card-header>
        <md-field>
          <md-field>
            <label>Username</label>
            <md-input v-model="accountData.username" disabled></md-input>
          </md-field>

          <md-field>
            <label>Name</label>
            <md-input v-model="accountData.name" disabled></md-input>
          </md-field>

          <md-field :class="{'md-invalid': errors.has('password')}" md-has-password>
            <label for="password">Type your wi-fi password</label>
            <md-input
              v-model="formData.password"
              name="password"
              type="password"
              v-validate="'required|min:8|password'"
              required
            />
            <span class="md-error">{{errors.first('password')}}</span>
          </md-field>

          <md-field :class="{'md-invalid': errors.has('password-confirm')}">
            <label>Confirm your wi-fi password</label>
            <md-input
              v-model="formData.confirmPassword"
              name="password-confirm"
              type="password"
              v-validate="'required|confirmed:password'"
              required
            />
            <span class="md-error">{{errors.first('password-confirm')}}</span>
          </md-field>
          <md-button type="submit" class="md-raised md-primary">Create VTEX Wi-Fi Account</md-button>
        </md-field>
      </md-card>
    </form>
  </div>
</template>

<script>
export default {
  name: 'Home',
  data: () => ({
    accountData: { username: '', name: '', role: '' },
    showProgress: false,
    formData: { password: '', confirmPassword: '', saveGSuitePassword: false }
  }),
  created: function() {
    this.setAccountDetail();
  },
  methods: {
    setAccountDetail() {
      var _this = this;
      this.$http
        .get(this.$apiPrefix + '/identity/account')
        .then(function(response) {
          console.info(
            'Account Detail. Status: OK, Body: ' + Object.keys(response.data)
          );
          _this.accountData = response.data;
          _this.processFormData(_this.accountData);
          _this.showProgress = false;
        })
        .catch(function(error) {
          _this.showProgress = false;
          if (error.response.status === 404) {
            if (_this.$isProduction) _this.$router.push('/create-account');
            else {
              _this.$router.push('/');
              _this.accountData = {
                username: 'alan.turing@vtex.com.br',
                name: 'Alan Turing',
                emails: ['alan.turing@vtex.com.br', 'alan.turing@vtex.com'],
                role: 'INTERNAL'
              };
            }
          } else if (error.response.status === 401) {
            _this.$auth.logout();
          } else {
            _this.notifyError({ message: error.response.data });
          }
          console.error(
            'VTEX Wi-Fi Account not found: ' + error.response.status
          );
        });
    },
    showSaveGSuitePasswordCheckbox() {
      return this.accountData.role === 'INTERNAL';
    },
    processFormData(accountData) {
      this.formData.saveGSuitePassword = accountData.saveGSuitePassword;
    },
    validateForm() {
      var _this = this;
      this.$validator
        .validateAll()
        .then(res => {
          if (!res) {
            // Catch errors
            console.warn('Form Invalid: ' + _this.errors);
            return;
          }
          _this.showProgress = true;
          console.info('Valid. Creating account');
          _this.$http
            .put(_this.$apiPrefix + '/identity/account', _this.formData)
            .then(function(response) {
              console.info('Account updated!' + response.data);
              _this.setAccountDetail();
              _this.notifyAccountUpdated();
              // Synchronize groups
              _this.$http
                .put(_this.$apiPrefix + '/identity/account/groups')
                .then(function(response) {
                  console.info('Groups synchronized!' + response.data);
                  _this.showProgress = false;
                  _this.notifyGroupsSynchronized();
                })
                .catch(function(error) {
                  console.warn('Error while synchronize groups! ' + error);
                  _this.showProgress = false;
                  var msgData = error.response.data;
                  _this.notifyError({
                    message:
                      typeof msgData === 'object' ? msgData.message : msgData
                  });
                });
            })
            .catch(function(error) {
              console.warn('Error while creating account! ' + error);
              _this.showProgress = false;
              var msgData = error.response.data;
              _this.notifyError({
                message: typeof msgData === 'object' ? msgData.message : msgData
              });
            });
        })
        .catch(function(e) {
          // Catch errors
          console.warn('Form Invalid: ' + e);
        });
    }
  },
  notifications: {
    notifyAccountUpdated: {
      title: 'Account updated',
      message: 'User account successfully updated.',
      type: 'success',
      timeout: 5000
    },
    notifyGroupsSynchronized: {
      title: 'Group synchronization',
      message: 'All groups synchronized.',
      type: 'success',
      timeout: 5000
    },
    notifyError: {
      title: 'Error Occured',
      type: 'error',
      timeout: 5000
    }
  }
};
</script>

<style>
.global-frame {
  max-width: 600px;
  margin: 5px;
  padding-left: 10px;
  padding-right: 5px;
}

.error-label {
  background-color: hotpink;
  color: black;
  padding: 5px;
}

.msg-label {
  background-color: darkseagreen;
  color: black;
  padding: 5px;
}

.md-gray {
  color: gray;
  font-size: 70%;
}

.md-primary {
  background-color: #f71963 !important;
  background-color: var(--md-theme-default-primary, #f71963);
}
</style>
