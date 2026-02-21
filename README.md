require("dotenv").config();
const express = require("express");
const mongoose = require("mongoose");
const bcrypt = require("bcryptjs");
const jwt = require("jsonwebtoken");
const fetch = require("node-fetch");
const multer = require("multer");
const fs = require("fs");
const cors = require("cors");

const app = express();
app.use(express.json());
app.use(cors());

/* ================= DATABASE ================= */
mongoose.connect(process.env.MONGO_URI);

const UserSchema = new mongoose.Schema({
  username: String,
  password: String
});

const ChatSchema = new mongoose.Schema({
  userId: String,
  messages: Array
});

const User = mongoose.model("User", UserSchema);
const Chat = mongoose.model("Chat", ChatSchema);

const upload = multer({ dest: "uploads/" });

/* ================= REGISTER ================= */
app.post("/register", async (req, res) => {
  const hashed = await bcrypt.hash(req.body.password, 10);
  await User.create({
    username: req.body.username,
    password: hashed
  });
  res.json({ msg: "تم التسجيل 👑" });
});

/* ================= LOGIN ================= */
app.post("/login", async (req, res) => {
  const user = await User.findOne({ username: req.body.username });
  if (!user) return res.json({ msg: "المستخدم غير موجود" });

  const valid = await bcrypt.compare(req.body.password, user.password);
  if (!valid) return res.json({ msg: "كلمة المرور خطأ" });

  const token = jwt.sign({ id: user._id }, process.env.JWT_SECRET);
  res.json({ token });
});

/* ================= CHAT ================= */
app.post("/chat", async (req, res) => {
  const token = req.headers.authorization;
  const decoded = jwt.verify(token, process.env.JWT_SECRET);

  let chat = await Chat.findOne({ userId: decoded.id });
  if (!chat) {
    chat = await Chat.create({ userId: decoded.id, messages: [] });
  }

  chat.messages.push({ role: "user", content: req.body.message });

  const response = await fetch("https://api.openai.com/v1/chat/completions", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "Authorization": `Bearer ${process.env.OPENAI_KEY}`
    },
    body: JSON.stringify({
      model: "gpt-4o-mini",
      messages: chat.messages
    })
  });

  const data = await response.json();
  const reply = data.choices[0].message.content;

  chat.messages.push({ role: "assistant", content: reply });
  await chat.save();

  res.json({ reply });
});

/* ================= IMAGE GENERATION ================= */
app.post("/generate-image", async (req, res) => {
  const response = await fetch("https://api.openai.com/v1/images/generations", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "Authorization": `Bearer ${process.env.OPENAI_KEY}`
    },
    body: JSON.stringify({
      model: "gpt-image-1",
      prompt: req.body.prompt,
      size: "1024x1024"
    })
  });

  const data = await response.json();
  res.json({ image: data.data[0].url });
});

/* ================= IMAGE ANALYSIS ================= */
app.post("/analyze-image", upload.single("image"), async (req, res) => {
  const imageBase64 = fs.readFileSync(req.file.path, { encoding: "base64" });

  const response = await fetch("https://api.openai.com/v1/chat/completions", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "Authorization": `Bearer ${process.env.OPENAI_KEY}`
    },
    body: JSON.stringify({
      model: "gpt-4o-mini",
      messages: [
        {
          role: "user",
          content: [
            { type: "text", text: "حلل الصورة دي" },
            {
              type: "image_url",
              image_url: {
                url: `data:image/jpeg;base64,${imageBase64}`
              }
            }
          ]
        }
      ]
    })
  });

  const data = await response.json();
  res.json({ analysis: data.choices[0].message.content });
});

app.listen(process.env.PORT, () =>
  console.log("🚀 LOMA EMPIRE RUNNING")
);
