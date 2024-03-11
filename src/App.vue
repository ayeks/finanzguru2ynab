<template>
  <div id="app" class="container">


    <div>
      <h3 class="title">Column Name Mapping</h3>
      <p>Change only if you want to map different collumns of the XLSX to the CSV files or if your export is not in German:</p>
    </div>
    <table>
      <thead>
        <tr>
          <th>Account</th>
          <th>Date</th>
          <th>Payee</th>
          <th>Memo</th>
          <th>Amount</th>
        </tr>
      </thead>
      <tbody>
        <tr>
          <td><input type="text" v-model="string_account" /></td>
          <td><input type="text" v-model="string_date" /></td>
          <td><input type="text" v-model="string_payee" /></td>
          <td><input type="text" v-model="string_memo" /></td>
          <td><input type="text" v-model="string_amount" /></td>
        </tr>
      </tbody>
    </table>
    
    <div>
      <label for="fileInput" class="button file-label">Select a Finanzguru XLSX file to generate YNAB CSV files</label>
      <input @change="handleFileUpload" class="hidden" type="file" id="fileInput" accept=".xlsx" hidden/>
    </div>

  </div>
</template>

<script>
import * as XLSX from "xlsx";
import * as Papa from "papaparse";

export default {
  data() {
    return {
      string_account: "Name Referenzkonto",
      string_date: "Buchungstag",
      string_payee: "Beguenstigter/Auftraggeber",
      string_memo: "Verwendungszweck",
      string_amount: "Betrag",
    };
  },
  methods: {

    handleFileUpload() {
      const fileInput = document.getElementById("fileInput");
      const file = fileInput.files[0];

      if (!file) {
        // Handle the case where no file is selected
        console.error("No file selected.");
        return;
      }

      this.readFile(file);
    },

    readFile(file) {
      const reader = new FileReader();

      reader.onload = (e) => {
        const data = new Uint8Array(e.target.result);
        this.processExcelData(data);
      };

      reader.readAsArrayBuffer(file);
    },

    processExcelData(data) {
      const workbook = XLSX.read(data, { type: "array" });
      const sheet = workbook.Sheets[workbook.SheetNames[0]];
      const jsonData = XLSX.utils.sheet_to_json(sheet, {
        raw: false, // This ensures that dates are parsed as JavaScript Date objects
      });

      // Extract headers
      const headers = Object.keys(jsonData[0]);

      // Set default values if not provided
      const {
        string_account,
        string_date,
        string_payee,
        string_memo,
        string_amount,
      } = this;

      // Rename columns
      headers.forEach((header) => {
        sheet[header] = this.renameColumn(sheet[header], {
          [string_account]: "Account",
          [string_date]: "Date",
          [string_payee]: "Payee",
          [string_memo]: "Memo",
          [string_amount]: "Amount",
        });
      });

      // Group by account using reduce
      const grouped = jsonData.reduce((acc, item) => {
        const account = item[string_account];
        if (!acc[account]) {
          acc[account] = [];
        }
        acc[account].push(item);
        return acc;
      }, {});


      // Create CSV files
      Object.entries(grouped).forEach(([name, group]) => {
        const subset = group.map((item) => ({
          Date: item[string_date],
          Payee: item[string_payee],
          Memo: item[string_memo],
          Amount: item[string_amount],
        }));
        this.downloadCSV(subset, name);
      });
    },

    renameColumn(column, mapping) {
      return mapping[column] || column;
    },

    downloadCSV(data, filename) {
      const csvContent = Papa.unparse(data);
      const blob = new Blob([csvContent], { type: "text/csv;charset=utf-8;" });

      const link = document.createElement("a");
      link.setAttribute("href", URL.createObjectURL(blob));
      link.setAttribute("download", `${filename}.csv`);
      document.body.appendChild(link);
      link.click();
      document.body.removeChild(link);
    },
  },
};
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}

button {
  margin-top: 20px;
  padding: 10px 20px;
  font-size: 16px;
}
</style>
