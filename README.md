# Zenquest.Zenchain.io-
How this website made, what's benifit provide this app, I provide full details how you can make this features website.

what the real site is doing (high level, with sources)
a runnable example frontend that shows the leaderboard and “connect wallet”,
a simple backend (Node/Express) that serves leaderboard data + totals,
DB schema and how to sign in with a wallet (signature-based),
notes on deploying and how to hook this up to the real ZenQuest endpoints if you want.

1) What the site does:-
From the page and docs the leaderboard shows: total participants and total XP, a ranked list (rank, username, XP), “Connect Wallet” prompt so users sign in with a web3 wallet, likely a single-page app that fetches leaderboard data from an API. 

2) Suggested tech stack (what I’d use to build this)
Frontend
React (Vite or Create-React-App)
ethers.js + web3modal/wagmi for wallet connect & signature
axios / fetch for API calls

Backend
Node.js + Express (or Next.js API routes)
Database: SQLite/Postgres (I’ll give examples with SQLite for simplicity)
Auth: wallet address + signed nonce (no passwords)

3) Frontend: React leaderboard + Connect Wallet (single file, plain JS + CSS)
Save as Leaderboard.jsx. This is a simple, self-contained React component that:
connects wallet via Web3Modal + ethers,
fetches /api/leaderboard and /api/stats,
signs a nonce for identity (example flow),
displays totals and ranked list.

// Leaderboard.jsx
import React, { useEffect, useState } from "react";
import { ethers } from "ethers";
import Web3Modal from "web3modal";
import axios from "axios";

export default function Leaderboard() {
  const [provider, setProvider] = useState(null);
  const [address, setAddress] = useState(null);
  const [connected, setConnected] = useState(false);
  const [stats, setStats] = useState({ participants: 0, totalXP: 0 });
  const [leaders, setLeaders] = useState([]);
  const [loading, setLoading] = useState(true);

  // fetch stats + leaderboard
  const fetchData = async () => {
    setLoading(true);
    try {
      const [sRes, lRes] = await Promise.all([
        axios.get("/api/stats"),
        axios.get("/api/leaderboard?limit=20"),
      ]);
      setStats(sRes.data);
      setLeaders(lRes.data);
    } catch (err) {
      console.error("fetch error", err);
    } finally {
      setLoading(false);
    }
  };

  useEffect(() => {
    fetchData();
  }, []);

  const connectWallet = async () => {
    try {
      const web3Modal = new Web3Modal({ cacheProvider: true });
      const instance = await web3Modal.connect();
      const prov = new ethers.providers.Web3Provider(instance);
      setProvider(prov);
      const signer = prov.getSigner();
      const addr = await signer.getAddress();
      setAddress(addr);
      setConnected(true);

      // OPTIONAL: sign a nonce/login flow
      const nonceRes = await axios.get(`/api/nonce?address=${addr}`);
      const nonce = nonceRes.data.nonce;
      const signature = await signer.signMessage(`Sign to login: ${nonce}`);
      // send signature to server for verification & session creation
      await axios.post("/api/login", { address: addr, signature });
      // refresh leaderboard (user-specific view may change)
      await fetchData();
    } catch (err) {
      console.error("wallet connect error", err);
    }
  };

  return (
    <div style={{ maxWidth: 900, margin: "2rem auto", fontFamily: "Inter, sans-serif" }}>
      <header style={{ display: "flex", justifyContent: "space-between", alignItems: "center" }}>
        <h1>Leaderboard</h1>
        {!connected ? (
          <button onClick={connectWallet} style={{ padding: "0.6rem 1rem", borderRadius: 8 }}>
            Connect Wallet
          </button>
        ) : (
          <div style={{ fontSize: 14 }}>
            Connected: {address.slice(0, 6)}...{address.slice(-4)}
          </div>
        )}
      </header>

      <section style={{ display: "flex", gap: "1.5rem", marginTop: "1rem" }}>
        <div style={{ flex: 1, padding: "1rem", borderRadius: 8, boxShadow: "0 2px 8px rgba(0,0,0,0.06)" }}>
          <div style={{ fontSize: 12, color: "#666" }}>Total Participants</div>
          <div style={{ fontSize: 28, fontWeight: 700 }}>{stats.participants ?? 0}</div>
        </div>
        <div style={{ flex: 1, padding: "1rem", borderRadius: 8, boxShadow: "0 2px 8px rgba(0,0,0,0.06)" }}>
          <div style={{ fontSize: 12, color: "#666" }}>Total XP Earned</div>
          <div style={{ fontSize: 28, fontWeight: 700 }}>{stats.totalXP ?? 0}</div>
        </div>
      </section>

      <main style={{ marginTop: "1.5rem" }}>
        <h2>Top Users</h2>
        {loading ? (
          <div>Loading...</div>
        ) : leaders.length === 0 ? (
          <div>No data</div>
        ) : (
          <table style={{ width: "100%", borderCollapse: "collapse", marginTop: 8 }}>
            <thead>
              <tr style={{ textAlign: "left", borderBottom: "1px solid #eee" }}>
                <th style={{ padding: "0.5rem 0" }}>Rank</th>
                <th>Username</th>
                <th>XP Points</th>
              </tr>
            </thead>
            <tbody>
              {leaders.map((u, i) => (
                <tr key={u.address} style={{ borderBottom: "1px solid #fafafa" }}>
                  <td style={{ padding: "0.6rem 0" }}>{i + 1}</td>
                  <td>{u.username ?? u.address}</td>
                  <td>{u.xp}</td>
                </tr>
              ))}
            </tbody>
          </table>
        )}
      </main>
    </div>
  );
}

Notes:
Install: npm i ethers web3modal axios.
Replace /api/* with your real API base URL.
This UI matches the features on the site (totals, list, connect wallet). Source: live page. 

4) Backend: Node + Express example (with SQLite)
This provides endpoints the frontend expects:
GET /api/stats → { participants, totalXP }
GET /api/leaderboard?limit=20 → array of top users
GET /api/nonce?address=0x... → returns a nonce string
POST /api/login → verify signature and create session (demo only)

// server.js
const express = require("express");
const bodyParser = require("body-parser");
const { ethers } = require("ethers");
const Database = require("better-sqlite3");
const cors = require("cors");

const db = new Database("leaderboard.db");
const app = express();
app.use(cors());
app.use(bodyParser.json());

// init DB (simple)
db.exec(`
CREATE TABLE IF NOT EXISTS users (
  address TEXT PRIMARY KEY,
  username TEXT,
  xp INTEGER DEFAULT 0,
  nonce TEXT
);
`);

// helper: get stats
app.get("/api/stats", (req, res) => {
  const participants = db.prepare("SELECT COUNT(*) AS c FROM users").get().c;
  const totalXP = db.prepare("SELECT COALESCE(SUM(xp),0) AS s FROM users").get().s;
  res.json({ participants, totalXP });
});

// leaderboard
app.get("/api/leaderboard", (req, res) => {
  const limit = parseInt(req.query.limit || "20", 10);
  const rows = db.prepare("SELECT address, username, xp FROM users ORDER BY xp DESC LIMIT ?").all(limit);
  res.json(rows);
});

// nonce
app.get("/api/nonce", (req, res) => {
  const address = (req.query.address || "").toLowerCase();
  if (!address) return res.status(400).json({ error: "no address" });
  const nonce = Math.random().toString(36).slice(2, 9);
  db.prepare("INSERT OR REPLACE INTO users(address, username, xp, nonce) VALUES (?, COALESCE((SELECT username FROM users WHERE address = ?), ?), COALESCE((SELECT xp FROM users WHERE address = ?), 0), ?)")
    .run(address, address, "Player", address, nonce);
  res.json({ nonce });
});

// login (verify signature)
app.post("/api/login", async (req, res) => {
  const { address, signature } = req.body;
  if (!address || !signature) return res.status(400).json({ error: "missing" });
  const row = db.prepare("SELECT nonce FROM users WHERE address = ?").get(address.toLowerCase());
  if (!row) return res.status(400).json({ error: "no user" });
  const message = `Sign to login: ${row.nonce}`;
  try {
    const signer = ethers.utils.verifyMessage(message, signature);
    if (signer.toLowerCase() === address.toLowerCase()) {
      // invalidate nonce
      db.prepare("UPDATE users SET nonce = ? WHERE address = ?").run(Math.random().toString(36).slice(2,9), address.toLowerCase());
      // create simple session token (for demo return address)
      res.json({ ok: true, address });
    } else {
      res.status(401).json({ error: "invalid signature" });
    }
  } catch (e) {
    res.status(400).json({ error: e.message });
  }
});

// demo route to increment XP (simulate quest completion)
app.post("/api/add-xp", (req, res) => {
  const { address, xp } = req.body;
  const r = db.prepare("UPDATE users SET xp = xp + ? WHERE address = ?").run(parseInt(xp||0,10), address.toLowerCase());
  if (r.changes === 0) {
    db.prepare("INSERT INTO users(address, username, xp) VALUES (?, ?, ?)").run(address.toLowerCase(), "Player", xp);
  }
  res.json({ ok: true });
});

app.listen(4000, () => console.log("API listening on :4000"));

Run: node server.js (install better-sqlite3, express, ethers, body-parser, cors).

5) DB / Schema
Simple users table (shown above) contains:address (PK) username xp (integer) nonce (for login signing)
You can extend with timestamps, referral codes, daily XP tracking, leaderboard snapshots, etc.

6) Security & best practices
Use signature-based login (nonce) to prove wallet ownership — no password stored. Rate-limit nonce and login endpoints.
Use HTTPS and secure session tokens (JWT or server sessions) after verifying signature.
For leaderboard integrity, only allow server-side XP updates when you verify the external action (e.g., a signed proof, on-chain event, or admin validation). Don’t let client increment XP directly without verification.
If using on-chain events (e.g., users earn XP when they make a tx), run a worker that listens to chain events and updates DB.

7) How to discover exactly how the real ZenQuest does it
If you want me to extract their actual API endpoints, response shapes, or frontend framework (to replicate it exactly), I can: inspect network requests used by https://zenquest.zenchain.io/leaderboard, fetch the JSON endpoints the page calls, and produce code that hits the real endpoints.


