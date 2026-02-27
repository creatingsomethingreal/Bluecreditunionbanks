const express = require('express');
const { Pool } = require('pg');
const cors = require('cors');

const app = express();

// Middleware
app.use(cors());
app.use(express.json());

// 1. Database Connection Logic
const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  ssl: {
    rejectUnauthorized: false // Required for Neon connection security
  }
});

// 2. API: Fetch All Members
app.get('/api/members', async (req, res) => {
  try {
    const result = await pool.query('SELECT * FROM users ORDER BY id ASC');
    res.json(result.rows);
  } catch (err) {
    res.status(500).json({ error: "Database error: " + err.message });
  }
});

// 3. API: Register New Member
app.post('/api/register', async (req, res) => {
  const { full_name, email, password } = req.body;
  // Generate a random 10-digit account number
  const accountNumber = Math.floor(1000000000 + Math.random() * 9000000000).toString();
  
  try {
    const newUser = await pool.query(
      'INSERT INTO users (full_name, email, password_hash, account_number, balance) VALUES ($1, $2, $3, $4, 0) RETURNING *',
      [full_name, email, password, accountNumber]
    );
    res.status(201).json(newUser.rows[0]);
  } catch (err) {
    res.status(500).json({ error: "Registration failed: " + err.message });
  }
});

// 4. API: Update Balance
app.put('/api/update-balance', async (req, res) => {
  const { id, balance } = req.body;
  try {
    await pool.query('UPDATE users SET balance = $1 WHERE id = $2', [balance, id]);
    res.json({ message: "Balance updated successfully" });
  } catch (err) {
    res.status(500).json({ error: "Update failed: " + err.message });
  }
});

// 5. API: Fetch Audit Logs
app.get('/api/logs', async (req, res) => {
  try {
    const result = await pool.query('SELECT * FROM transactions ORDER BY created_at DESC');
    res.json(result.rows);
  } catch (err) {
    res.status(500).json({ error: "Could not fetch logs" });
  }
});

// Export for Vercel
module.exports = app;