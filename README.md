# Smart-Parking-Lot-System


Low-Level Architecture (Node.js + MongoDB)

### 🧱 1. **Tech Stack**

- **Backend**: Node.js (Express)
- **Database**: MongoDB
- **Concurrency Handling**: MongoDB Transactions (or optimistic locking)
- **Time**: `Date.now()` for timestamping

### **🧩 2. MongoDB Data Models (Schemas)**

**🔹 `floors` collection**

```jsx
{
	_id: ObjectId,
	floorNumber: Number,
	totalSpots: Number,
	availableSpots: Number
}
```

**🔹 `spots` collection**

```jsx
	{
		_id: ObjectId,
		floorId: ObjectId,
		spotNumber: Number,
		size: "MOTORCYCLE" | "CAR" | "BUS",
		isAvailable: Boolean
	}	
	
```

**🔹 `vehicles` collection**

```jsx
{
	_id: ObjectId,
	licensePlate: String,
	type: "MOTORCYCLE" | "CAR" | "BUS"
}
```

**🔹 `transactions` collection**

```jsx
{
	_id: ObjectId,
	vehicleId: ObjectId,
	spotId: ObjectId,
	entryTime: Date,
	exitTime: Date,
	fee: Number
}
```

### 🔁 3. Parking Spot Allocation Logic

```jsx
async function allocateSpot(vehicleType) {
  return await Spot.findOneAndUpdate(
    { size: vehicleType, isAvailable: true },
    { $set: { isAvailable: false } },
    { sort: { floorId: 1, spotNumber: 1 }, new: true }
  );
}
```

### 🚗 4. Check-In API

**POST `/api/checkin`**

// Sample request: { licensePlate: "ABC123", type: "CAR" }

```jsx
app.post("/api/checkin", async (req, res) => {
  const { licensePlate, type } = req.body;

  const vehicle = await Vehicle.findOneAndUpdate(
    { licensePlate },
    { $setOnInsert: { type } },
    { upsert: true, new: true }
  );

  const spot = await allocateSpot(type);
  if (!spot) return res.status(400).json({ error: "No available spot" });

  const transaction = await Transaction.create({
    vehicleId: vehicle._id,
    spotId: spot._id,
    entryTime: new Date(),
  });

  res.json({
    spotId: spot._id,
    floorId: spot.floorId,
    spotNumber: spot.spotNumber,
  });
});
```

### 🚙 5. Check-Out API

**POST `/api/checkout`**

```jsx
app.post("/api/checkout", async (req, res) => {
  const { licensePlate } = req.body;
  const vehicle = await Vehicle.findOne({ licensePlate });
  if (!vehicle) return res.status(404).json({ error: "Vehicle not found" });

  const transaction = await Transaction.findOne({
    vehicleId: vehicle._id,
    exitTime: { $exists: false },
  });

  if (!transaction)
    return res.status(400).json({ error: "No active parking session" });

  const durationMs = Date.now() - transaction.entryTime.getTime();
  const fee = calculateFee(vehicle.type, durationMs);

  transaction.exitTime = new Date();
  transaction.fee = fee;
  await transaction.save();

  await Spot.findByIdAndUpdate(transaction.spotId, {
    $set: { isAvailable: true },
  });

  res.json({ durationMinutes: Math.ceil(durationMs / 60000), fee });
});

function calculateFee(type, durationMs) {
  const rates = { MOTORCYCLE: 10, CAR: 20, BUS: 50 };
  const hours = Math.ceil(durationMs / (60 * 60 * 1000));
  return rates[type] * hours;
}
```

### 🧷 7. Concurrency Handling

MongoDB handles atomic updates on single documents. To manage **simultaneous check-ins**, we:

- Use **`findOneAndUpdate`** to ensure atomic spot allocation.
