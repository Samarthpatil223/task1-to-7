# task1-to-6



#task_01

const express = require('express');
const axios = require('axios');
const mongoose = require('mongoose');

const app = express();

// Connect to MongoDB
mongoose.connect('mongodb://localhost:27017/transactions', { useNewUrlParser: true, useUnifiedTopology: true });

const TransactionSchema = new mongoose.Schema({
  id: Number,
  title: String,
  description: String,
  price: Number,
  category: String,
  sold: Boolean,
  dateOfSale: Date
});

const Transaction = mongoose.model('Transaction', TransactionSchema);

// API to initialize the database
app.get('/api/init', async (req, res) => {
  try {
    const response = await axios.get('https://s3.amazonaws.com/roxiler.com/product_transaction.json');
    const data = response.data;

    // Insert data into the database
    await Transaction.insertMany(data);
    res.status(200).json({ message: 'Database initialized with seed data' });
  } catch (error) {
    res.status(500).json({ error: 'Error fetching data from API' });
  }
});

app.listen(3000, () => {
  console.log('Server is running on port 3000');

});

#task_02

app.get('/api/transactions', async (req, res) => {
  const { search, page = 1, perPage = 10 } = req.query;
  let query = {};
  
  // Search logic
  if (search) {
    query = {
      $or: [
        { title: new RegExp(search, 'i') },
        { description: new RegExp(search, 'i') },
        { price: new RegExp(search, 'i') }
      ]
    };
  }
  
  // Pagination logic
  const skip = (page - 1) * perPage;
  
  try {
    const transactions = await Transaction.find(query).skip(skip).limit(parseInt(perPage));
    res.status(200).json(transactions);
  } catch (error) {
    res.status(500).json({ error: 'Error fetching transactions' });
  }
});

#task_03

app.get('/api/statistics', async (req, res) => {
  const { month } = req.query;
  try {
    const transactions = await Transaction.find({
      dateOfSale: { $regex: `-${month.padStart(2, '0')}-` }
    });
    const totalSale = transactions.reduce((acc, curr) => acc + curr.price, 0);
    const soldItems = transactions.filter(t => t.sold).length;
    const unsoldItems = transactions.filter(t => !t.sold).length;
    
    res.status(200).json({
      totalSale,
      soldItems,
      unsoldItems
    });
  } catch (error) {
    res.status(500).json({ error: 'Error calculating statistics' });
  }
});

#task_04

app.get('/api/bar-chart', async (req, res) => {
  const { month } = req.query;
  const priceRanges = {
    "0-100": 0, "101-200": 0, "201-300": 0, "301-400": 0,
    "401-500": 0, "501-600": 0, "601-700": 0, "701-800": 0,
    "801-900": 0, "901-above": 0
  };
  
  try {
    const transactions = await Transaction.find({
      dateOfSale: { $regex: `-${month.padStart(2, '0')}-` }
    });
    transactions.forEach(t => {
      if (t.price <= 100) priceRanges["0-100"]++;
      else if (t.price <= 200) priceRanges["101-200"]++;
      // Add remaining ranges logic here...
      else priceRanges["901-above"]++;
    });
    res.status(200).json(priceRanges);
  } catch (error) {
    res.status(500).json({ error: 'Error generating bar chart data' });
  }
});

#task_05

app.get('/api/pie-chart', async (req, res) => {
  const { month } = req.query;
  
  try {
    const transactions = await Transaction.find({
      dateOfSale: { $regex: `-${month.padStart(2, '0')}-` }
    });
    const categories = {};
    transactions.forEach(t => {
      categories[t.category] = (categories[t.category] || 0) + 1;
    });
    res.status(200).json(categories);
  } catch (error) {
    res.status(500).json({ error: 'Error generating pie chart data' });
  }
});

#task_06

app.get('/api/combined', async (req, res) => {
  const { month } = req.query;
  
  try {
    const stats = await axios.get(`http://localhost:3000/api/statistics?month=${month}`);
    const barChart = await axios.get(`http://localhost:3000/api/bar-chart?month=${month}`);
    const pieChart = await axios.get(`http://localhost:3000/api/pie-chart?month=${month}`);
    
    res.status(200).json({
      statistics: stats.data,
      barChart: barChart.data,
      pieChart: pieChart.data
    });
  } catch (error) {
    res.status(500).json({ error: 'Error fetching combined data' });
  }
});


