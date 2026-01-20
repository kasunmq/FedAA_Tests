# Modification Plan: Save Client-Level Data to clients_<Date>.txt

## Overview
You want to save client-level data at the end of each round to a file named `clients_<Date>.txt` with the following columns:
- **Client Number** - Client ID
- **Loss Value** - Training loss of the client
- **Selection Count** - Number of times client was selected for training
- **Aggregation Count** - Number of times client was selected for aggregation

---

## Areas to Modify

### 1. **File: `servers/serverbase.py`**
   **Location:** `__init__` method (after result files initialization)
   
   **What to Add:** Initialize client-level tracking dictionaries and create clients results file
   
   **Current Code (should find these lines around line 60-80):**
   ```python
   # Initialize result files for global and local models
   # ... existing code ...
   with open(self.local_results_file, 'w') as f:
       f.write(header)
   ```
   
   **After Modification (ADD these lines after the above section):**
   ```python
   # Initialize client-level tracking
   self.client_losses = {i: 0.0 for i in range(self.num_clients)}  # Latest loss per client
   self.client_selection_count = {i: 0 for i in range(self.num_clients)}  # Times selected for training
   self.client_aggregation_count = {i: 0 for i in range(self.num_clients)}  # Times selected for aggregation
   
   # Create clients results file
   self.clients_results_file = os.path.join(self.results_dir, f"clients_{time_now}.txt")
   client_header = "Client Number,Loss Value,Selection Count,Aggregation Count\n"
   with open(self.clients_results_file, 'w') as f:
       f.write(client_header)
   ```

---

### 2. **File: `servers/serverbase.py`**
   **Location:** Add a new method called `save_client_round_data()`
   
   **What to Add:** Create a new method that saves client-level data at the end of each round
   
   **Add this new method (can be added after the `evaluate()` method):**
   ```python
   def save_client_round_data(self, round_num):
       """Save client-level data for the current round"""
       with open(self.clients_results_file, 'a') as f:
           for client_id in range(self.num_clients):
               loss = self.client_losses.get(client_id, 0.0)
               selection_count = self.client_selection_count.get(client_id, 0)
               aggregation_count = self.client_aggregation_count.get(client_id, 0)
               # Format: Client Number,Loss Value,Selection Count,Aggregation Count
               f.write(f"{client_id},{loss:.4f},{selection_count},{aggregation_count}\n")
   ```

---

### 3. **File: `servers/serverFedAA.py`**
   **Location:** `train()` method - client training loop (around line 80-85)
   
   **What to Add:** Track when clients are selected for training by updating `client_selection_count`
   
   **Current Code (lines ~80-85):**
   ```python
   train_time_begin = time.time()
   for client in self.clients:
       client.train()
       if client.id in self.ad_clients:
   ```
   
   **After Modification (UPDATE the loop):**
   ```python
   train_time_begin = time.time()
   for client in self.clients:
       client.train()
       # Track client selection and loss
       self.client_selection_count[client.id] += 1
       self.client_losses[client.id] = getattr(client, 'train_loss', 0.0)
       if client.id in self.ad_clients:
   ```

---

### 4. **File: `servers/serverFedAA.py`**
   **Location:** `get_state()` method (around line 35)
   
   **What to Add:** Track when clients are selected for aggregation by updating `client_aggregation_count`
   
   **Current Code (lines ~35-45):**
   ```python
   def get_state(self):
       random_indices = torch.tensor(random.sample(range(0, self.num_clients), self.num_sel_clients))
       # ... more code ...
       _, indices = torch.topk(state_temp_sum, k=self.aggre_num, largest=False)
       state = state_temp_sum[indices]
       state /= state.max()
       self.aggre_clients = [self.selected_clients[i] for i in indices]
       return state, self.aggre_clients
   ```
   
   **After Modification (UPDATE the return section):**
   ```python
   def get_state(self):
       random_indices = torch.tensor(random.sample(range(0, self.num_clients), self.num_sel_clients))
       # ... more code ...
       _, indices = torch.topk(state_temp_sum, k=self.aggre_num, largest=False)
       state = state_temp_sum[indices]
       state /= state.max()
       self.aggre_clients = [self.selected_clients[i] for i in indices]
       
       # Track aggregation count
       for aggre_client in self.aggre_clients:
           self.client_aggregation_count[aggre_client.id] += 1
       
       return state, self.aggre_clients
   ```

---

### 5. **File: `servers/serverFedAA.py`**
   **Location:** `train()` method - end of each round (after local evaluation, around line 105-110)
   
   **What to Add:** Call the method to save client-level data at the end of each round
   
   **Current Code (around line 105-115):**
   ```python
   print("\nEvaluate local model")
   self.evaluate(self.benign_clients, round_num=i, model_type='local')

   next_state, self.aggre_clients = self.get_state()

   self.replay_buffer.add(state, action, next_state, reward, done)
   ```
   
   **After Modification (ADD after local evaluation):**
   ```python
   print("\nEvaluate local model")
   self.evaluate(self.benign_clients, round_num=i, model_type='local')
   
   # Save client-level data for this round
   self.save_client_round_data(i)

   next_state, self.aggre_clients = self.get_state()

   self.replay_buffer.add(state, action, next_state, reward, done)
   ```

---

## Summary of Changes

| File | Method/Location | Change Type | What |
|------|-----------------|-------------|------|
| serverbase.py | `__init__()` | ADD | Initialize client tracking dicts and create clients file |
| serverbase.py | (new method) | ADD | Create `save_client_round_data()` method |
| serverFedAA.py | `train()` - client loop | MODIFY | Track selection count and store loss (lines ~83) |
| serverFedAA.py | `get_state()` | ADD | Track aggregation count (lines ~45) |
| serverFedAA.py | `train()` - round end | ADD | Call `save_client_round_data(i)` (after line ~105) |

---

## File Output Structure

After running, you'll get:
```
FedAA/
├── RESULTS/
│   ├── global_2026-01-21_15-30-45.txt
│   ├── local_2026-01-21_15-30-45.txt
│   └── clients_2026-01-21_15-30-45.txt
├── servers/
├── clients/
└── main.py
```

**File Format (clients_<Date>.txt):**
```
Client Number,Loss Value,Selection Count,Aggregation Count
0,0.0234,5,2
1,0.0178,4,3
2,0.0312,6,1
3,0.0245,3,4
4,0.0198,5,2
...
(100 rows for 100 clients - one row per client per round)
```

---

## Data Interpretation

After **Round 0**, the file will have **100 rows** (one per client)
After **Round 1**, the file will have **200 rows** (100 clients × 2 rounds)
After **Round N**, the file will have **(N+1) × 100 rows**

You can then analyze:
- Which clients have highest/lowest loss
- Which clients are selected most frequently for training
- Which clients are selected most frequently for aggregation
- Patterns in client selection over rounds

---

## Notes

✅ **What stays the same:**
- All console print statements remain unchanged
- Global and local model result files continue as before
- No breaking changes

✅ **What's new:**
- Creates `clients_<Date>.txt` file in RESULTS folder
- Tracks loss for every client during training
- Tracks how many times each client is selected for training
- Tracks how many times each client is selected for aggregation
- Saves all this data after each round

✅ **Safe to implement:**
- Uses existing data structures and methods
- No logic changes to training algorithm
- Can be added without affecting performance

---

## Key Points

1. **Selection Count** will increase by 1 for each client that calls `train()` in that round
2. **Aggregation Count** will increase by 1 for each client in `self.aggre_clients` 
3. **Loss Value** will be the latest training loss from that client
4. Data is appended to file after each round completes

---

**Ready to implement when you confirm!**
