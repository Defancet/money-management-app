<!--
    Name: components/TransactionsPage.vue
    Authors: Volodymyr Burylov
             Volodymyr Kaznacheiev
             Maksim Kalutski
    Date: 05/12/2023
-->

<template>
  <div class="add-transaction-button">
    <button class="btn btn-primary" type="submit" @click="show_add_transaction_modal = !show_add_transaction_modal">
      <span class="material-symbols-outlined">add</span>
      Add new transaction
    </button>
    <Modal name="m1" v-model:visible="show_add_transaction_modal" :modalClass="'custom-modal'"
           :title="editMode ? 'Edit Transaction' : 'Add Transaction'" :maskClosable="false" :closable="false"
           :cancelButton="{text: 'Cancel', onclick: () => { cancelEditTransaction() }, loading: false}"
           :okButton="{text: editMode ? 'Save Changes' : 'Add Transaction +', onclick: () => { editMode ? save_edited_transaction() : add_transaction() }, loading: false}">
      <div>
        <div class="form-group">
          <div class="display-type-buttons-transaction">
            <div class="display-type-buttons-container-transaction">
              <button type="button"
                      :class="[!transaction_is_income ? 'btn-display-type-selected-transaction' : 'btn-display-type-transaction']"
                      @click="transaction_is_income = false; transaction_category_id = 0">Expense
              </button>
              <button type="button"
                      :class="[transaction_is_income ? 'btn-display-type-selected-transaction' : 'btn-display-type-transaction']"
                      @click="transaction_is_income = true; transaction_category_id = 0">Income
              </button>
            </div>
          </div>
          <br/>
          <select class="form-select" aria-label="Select transaction" v-model="transaction_category_id">
            <option :value="0" selected>Select transaction category</option>
            <option v-if="transaction_is_income" v-for="category in categories.filter(property => property.isIncome)"
                    :value="category.id">{{ category.name }}
            </option>
            <option v-if="!transaction_is_income" v-for="category in categories.filter(property => !property.isIncome)"
                    :value="category.id">{{ category.name }}
            </option>
          </select>
          <div class="input-group my-3">
            <input type="text" class="form-control" v-model="transaction_description" placeholder="Input description"/>
          </div>
          <div class="input-group my-3">
            <input type="date" class="form-control" v-model="transaction_date" placeholder="Transaction date"/>
          </div>
          <div class="input-group my-3">
            <input type="number" class="form-control" v-model="transaction_amount" placeholder="Input amount $"/>
          </div>
        </div>
      </div>
    </Modal>
  </div>
  <div class="transaction-table-container">
    <div class="row">
      <div class="col">
        <table class="table">
          <thead>
          <tr class="table-secondary">
            <th scope="col"></th>
            <th scope="col">Category</th>
            <th scope="col">Description</th>
            <th scope="col">Date</th>
            <th scope="col">Amount</th>
            <th scope="col"></th>
          </tr>
          </thead>
          <tbody>
          <tr v-for="transaction in paginatedTransactions" :key="transaction.id">
            <td>
              <span v-if="transaction.isIncome" class="material-symbols-outlined" style="color: green;">
                keyboard_double_arrow_up
              </span>
              <span v-else class="material-symbols-outlined" style="color: red">
                keyboard_double_arrow_down
              </span>
            </td>
            <td>{{ get_category_name_by_id(transaction.categoryId) }}</td>
            <td>{{ transaction.description }}</td>
            <td>{{ formatISODateToDateTime(transaction.date) }}</td>
            <td>{{ transaction.amount }} $</td>
            <td>
              <button @click="edit_transaction(transaction.id)" class="btn-edit">
                <span class="material-symbols-outlined">edit</span>
              </button>
              <button @click="confirm_delete_transaction(transaction.id)" class="btn-delete">
                <span class="material-symbols-outlined">delete</span>
              </button>
            </td>
          </tr>
          </tbody>
        </table>
      </div>
    </div>
  </div>
  <div class="pagination">
    <p class="pagination_text">Page {{ currentPage }} / {{ totalPages }}</p>
    <button @click="prevPage" :disabled="currentPage === 1" class="pagination-button">&lt;</button>
    <button @click="nextPage" :disabled="currentPage === totalPages" class="pagination-button">&gt;</button>
  </div>
</template>

<script setup>
import axios from 'axios';
import {ref, computed, onMounted} from 'vue';
import {Modal} from 'usemodal-vue3';
import {format} from 'date-fns';

const sort = ref('');
const transaction_category_id = ref(0);
const transaction_description = ref('');
const transaction_date = ref(new Date().toISOString().slice(0, 10));
const transaction_amount = ref();
const transaction_is_income = ref(false);
const transactions = ref([]);
const categories = ref([]);
const show_add_transaction_modal = ref(false);
const editMode = ref(false);
const selectedTransactionId = ref(null);

onMounted(() => {
  get_categories_data();
  get_transactions_data();
});

function add_transaction() {
  show_add_transaction_modal.value = !show_add_transaction_modal.value;
  save_transaction_to_db();
}

function save_transaction_to_db() {
  axios.post('http://localhost:3000/add_transaction', null, {
    params: {
      category_id: transaction_category_id.value,
      description: transaction_description.value,
      date: transaction_date.value,
      amount: transaction_amount.value,
      is_income: transaction_is_income.value
    }
  })
      .then(response => {
        get_transactions_data();
        reset_transaction_form();
      }).catch(error => {
    handle_error(error);
  });
}

function reset_transaction_form() {
  transaction_category_id.value = 0;
  transaction_description.value = '';
  transaction_date.value = new Date().toISOString().slice(0, 10);
  transaction_amount.value = '';
  transaction_is_income.value = false;
}

function handle_error(error) {
  if (error.response && error.response.data && error.response.data.error) {
    alert(error.response.data.error);
  } else {
    alert('An error occurred: ' + error.message);
  }
}

function get_transactions_data() {
  axios.get('http://localhost:3000/get_transactions')
      .then(response => {
        transactions.value = response.data;
      });
}

function get_categories_data() {
  axios.get('http://localhost:3000/get_categories')
      .then(response => {
        categories.value = response.data;
      });
}

function confirm_delete_transaction(id) {
  if (confirm('Are you sure you want to delete this transaction?')) {
    delete_transaction(id);
  }
}

function delete_transaction(id) {
  axios.delete('http://localhost:3000/delete_transaction', {
    params: {
      id: id
    }
  })
      .then(response => {
        get_transactions_data();
      });
}

function get_category_name_by_id(id) {
  let elem = categories.value.find(category => category.id == id);
  return elem ? elem.name : '';
}

function setSort(value) {
  sort.value = value;
}

const filteredTransactions = computed(() => {
  switch (sort.value) {
    case 'Week':
      return filterTransactionsByWeek(transactions.value);
    case 'Month':
      return filterTransactionsByMonth(transactions.value);
    case 'Year':
      return filterTransactionsByYear(transactions.value);
    default:
      return transactions.value;
  }
});

function filterTransactionsByWeek(transactions) {
  const now = new Date();
  const oneWeekAgo = new Date(now.getTime() - 7 * 24 * 60 * 60 * 1000);
  return transactions.filter(transaction => {
    const transactionDate = new Date(transaction.date);
    return transactionDate >= oneWeekAgo && transactionDate <= now;
  });
}

function filterTransactionsByMonth(transactions) {
  const now = new Date();
  const oneMonthAgo = new Date(now.getFullYear(), now.getMonth() - 1, now.getDate());
  return transactions.filter(transaction => {
    const transactionDate = new Date(transaction.date);
    return transactionDate >= oneMonthAgo && transactionDate <= now;
  });
}

function filterTransactionsByYear(transactions) {
  const now = new Date();
  const oneYearAgo = new Date(now.getFullYear() - 1, now.getMonth(), now.getDate());
  return transactions.filter(transaction => {
    const transactionDate = new Date(transaction.date);
    return transactionDate >= oneYearAgo && transactionDate <= now;
  });
}

function formatISODateToDateTime(isoDateString) {
  const dateTime = new Date(isoDateString);
  return format(dateTime, 'dd-MM-yyyy HH:mm');
}

const itemsPerPage = ref(20);
const currentPage = ref(1);

const nextPage = () => {
  if (currentPage.value < totalPages.value) {
    currentPage.value += 1;
  }
};

const prevPage = () => {
  if (currentPage.value > 1) {
    currentPage.value -= 1;
  }
};

const totalPages = computed(() => {
  return Math.ceil(filteredTransactions.value.length / itemsPerPage.value);
});

const paginatedTransactions = computed(() => {
  const start = (currentPage.value - 1) * itemsPerPage.value;
  const end = start + itemsPerPage.value;
  return filteredTransactions.value.slice(start, end);
});

function edit_transaction(id) {
  const selectedTransaction = transactions.value.find(transaction => transaction.id === id);
  if (selectedTransaction) {
    editMode.value = true;
    show_add_transaction_modal.value = true;
    selectedTransactionId.value = id;
    transaction_category_id.value = selectedTransaction.categoryId;
    transaction_description.value = selectedTransaction.description;
    transaction_date.value = selectedTransaction.date.slice(0, 10);
    transaction_amount.value = selectedTransaction.amount;
    transaction_is_income.value = selectedTransaction.isIncome;
  }
}

function cancelEditTransaction() {
  editMode.value = false;
  show_add_transaction_modal.value = false;
}

function save_edited_transaction() {
  editMode.value = false;
  show_add_transaction_modal.value = false;
  const id = selectedTransactionId.value;
  axios.put('http://localhost:3000/edit_transaction', null, {
    params: {
      id: id,
      category_id: transaction_category_id.value,
      description: transaction_description.value,
      date: transaction_date.value,
      amount: transaction_amount.value,
      is_income: transaction_is_income.value
    }
  })
      .then(response => {
        get_transactions_data();
      }).catch(error => {
    handle_error(error);
  });
}
</script>

<style scoped>
.title {
  text-align: left;
  margin-left: 30px;
  margin-top: 30px;
  color: rgba(0, 0, 0, 0.5);
}

.add-transaction-button {
  display: flex;
  align-items: left;
  margin-top: 1.5vw;
}

.add-transaction-button .btn {
  display: flex;
  align-items: center;
  margin-left: auto;
  margin-right: 40px;
  font-size: 16px;
}

.display-type-buttons-transaction {
  display: flex;
  justify-content: center;
}

.display-type-buttons-container-transaction {
  background-color: #edeef2;
  border-radius: 20px;
}

.btn-display-type-transaction {
  background-color: #edeef2;
  border: none;
  width: 200px;
  padding: 10px;
  border-radius: 20px;
  height: 40px;
  font-weight: normal;
  font-size: 15px;
  transition: background-color 0.5s ease;
}

.btn-display-type-transaction:hover {
  background-color: #e0e1e6;
}

.btn-display-type-selected-transaction {
  background-color: #C6E7FF;
  border: none;
  width: 200px;
  padding: 10px;
  border-radius: 20px;
  height: 40px;
  font-weight: normal;
  font-size: 15px;
}

.transaction-table-container {
  margin-left: 30px;
  margin-top: 20px;
}

.btn-edit, .btn-delete {
  background: none;
  border: none;
  cursor: pointer;
}

.pagination {
  display: flex;
  float: right;
  justify-content: center;
  margin-bottom: 1vw;
  margin-right: 3vw;
}

.pagination_text {
  margin-right: 10px;
  margin-top: 15px;
  font-size: 16px;
  color: rgba(0, 0, 0, 0.5);
}

.pagination-button {
  background: none;
  border: none;
  font-size: 24px;
}

.custom-modal {
  border-radius: 50%;
}
</style>