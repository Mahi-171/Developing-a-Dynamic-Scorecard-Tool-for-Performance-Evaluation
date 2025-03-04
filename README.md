# Developing-a-Dynamic-Scorecard-Tool-for-Performance-Evaluation

#server.js

#javascript

require("dotenv").config();
const express = require("express");
const mongoose = require("mongoose");
const cors = require("cors");
const bodyParser = require("body-parser");

const app = express();
app.use(cors());
app.use(bodyParser.json());

mongoose.connect(process.env.MONGO_URI, { useNewUrlParser: true, useUnifiedTopology: true })
    .then(() => console.log("MongoDB Connected"))
    .catch(err => console.log(err));

// Import Routes
const scoreRoutes = require("./routes/scoreRoutes");
const authRoutes = require("./routes/authRoutes");

app.use("/api/scores", scoreRoutes);
app.use("/api/auth", authRoutes);

const PORT = process.env.PORT || 5000;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));

#2. Database Schema
models/Score.js

#javascript
const mongoose = require("mongoose");

const ScoreSchema = new mongoose.Schema({
    user: { type: mongoose.Schema.Types.ObjectId, ref: "User" },
    category: String,
    productivity: Number,
    quality: Number,
    timeliness: Number,
    totalScore: Number,
}, { timestamps: true });

module.exports = mongoose.model("Score", ScoreSchema);
models/User.js


const mongoose = require("mongoose");

const UserSchema = new mongoose.Schema({
    name: String,
    email: { type: String, unique: true },
    password: String,
    role: { type: String, enum: ["admin", "viewer"], default: "viewer" }
});

module.exports = mongoose.model("User", UserSchema);


#3. Score Calculation API
routes/scoreRoutes.js

javascript
Copy code
const express = require("express");
const Score = require("../models/Score");

const router = express.Router();

// Add Score
router.post("/", async (req, res) => {
    try {
        const { user, category, productivity, quality, timeliness } = req.body;
        const totalScore = (productivity * 0.4) + (quality * 0.4) + (timeliness * 0.2);

        const newScore = new Score({ user, category, productivity, quality, timeliness, totalScore });
        await newScore.save();
        res.json(newScore);
    } catch (err) {
        res.status(500).send("Server Error");
    }
});

// Fetch All Scores
router.get("/", async (req, res) => {
    try {
        const scores = await Score.find().populate("user", "name email");
        res.json(scores);
    } catch (err) {
        res.status(500).send("Server Error");
    }
});

module.exports = router;

#4. Authentication API
routes/authRoutes.js

const express = require("express");
const bcrypt = require("bcryptjs");
const jwt = require("jsonwebtoken");
const User = require("../models/User");

const router = express.Router();

// Register
router.post("/register", async (req, res) => {
    try {
        const { name, email, password, role } = req.body;
        const hashedPassword = await bcrypt.hash(password, 10);

        const newUser = new User({ name, email, password: hashedPassword, role });
        await newUser.save();
        res.json("User registered successfully");
    } catch (err) {
        res.status(500).send("Server Error");
    }
});

// Login
router.post("/login", async (req, res) => {
    try {
        const { email, password } = req.body;
        const user = await User.findOne({ email });
        if (!user) return res.status(400).json("Invalid Credentials");

        const isMatch = await bcrypt.compare(password, user.password);
        if (!isMatch) return res.status(400).json("Invalid Credentials");

        const token = jwt.sign({ id: user._id, role: user.role }, process.env.JWT_SECRET, { expiresIn: "1h" });
        res.json({ token, user });
    } catch (err) {
        res.status(500).send("Server Error");
    }
});


module.exports = router;
#5. Frontend: React.js

Install Dependencies

sh
Copy code
npx create-react-app client
cd client
npm install axios react-chartjs-2 chart.js
App.js


import React, { useState, useEffect } from "react";
import axios from "axios";
import Chart from "react-chartjs-2";

const App = () => {
    const [scores, setScores] = useState([]);

    useEffect(() => {
        axios.get("http://localhost:5000/api/scores")
            .then(res => setScores(res.data))
            .catch(err => console.log(err));
    }, []);

    return (
        <div>
            <h1>Performance Scorecard</h1>
            <table border="1">
                <thead>
                    <tr>
                        <th>User</th>
                        <th>Category</th>
                        <th>Productivity</th>
                        <th>Quality</th>
                        <th>Timeliness</th>
                        <th>Total Score</th>
                    </tr>
                </thead>
                <tbody>
                    {scores.map(score => (
                        <tr key={score._id}>
                            <td>{score.user?.name}</td>
                            <td>{score.category}</td>
                            <td>{score.productivity}</td>
                            <td>{score.quality}</td>
                            <td>{score.timeliness}</td>
                            <td>{score.totalScore}</td>
                        </tr>
                    ))}
                </tbody>
            </table>
            <Chart
                type="bar"
                data={{
                    labels: scores.map(s => s.category),
                    datasets: [{
                        label: "Total Score",
                        data: scores.map(s => s.totalScore),
                        backgroundColor: "rgba(75, 192, 192, 0.6)"
                    }]
                }}
            />
        </div>
    );
};

export default App;








