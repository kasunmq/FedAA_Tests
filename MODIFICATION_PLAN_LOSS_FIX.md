# Modification Plan: Capture Individual Client Losses from `train_metrics()`

**Date**: 21 January 2026  
**Status**: IMPLEMENTATION IN PROGRESS

---

## **Problem Statement**

Client-level loss values in `clients_<Date>.txt` are all showing 0.0000 instead of actual loss values:

```
Client Number,Loss Value,Selection Count,Aggregation Count
0,0.0000,10,0
1,0.0000,10,0
2,0.0000,10,0
```

**Root Cause**: `self.train_loss` is never set in the client's `train()` method, so the server gets the default value (0.0) via `getattr(client, 'train_loss', 0.0)`.

---

## **Solution Overview**

Instead of modifying the client code, reuse loss values already being calculated in the `train_metrics()` method during evaluation. This method already computes individual client losses accurately - we just need to store them before aggregating.

---

## **Benefits**

✅ Uses loss values already calculated in `train_metrics()`  
✅ No modification needed to `clientFedAA.py`  
✅ Captures actual loss from training data re-evaluation  
✅ Different clients can show different losses (realistic)  
✅ Minimal code change (2-3 lines)  

---

## **Implementation Details**

### **File Modified**
- **Path**: `/Users/kasunerandawijethilake/PhD/FED_AA_2/FedAA/servers/serverbase.py`
- **Method**: `train_metrics()` (lines 232-243)

### **Change Made**

**Before:**
```python
def train_metrics(self, benign_clients):
    num_samples = []
    losses = []
    for c in self.clients:
        if c.id in benign_clients:
            cl, ns = c.train_metrics()
            num_samples.append(ns)
            losses.append(cl * 1.0)

    ids = benign_clients

    return ids, num_samples, losses
```

**After:**
```python
def train_metrics(self, benign_clients):
    num_samples = []
    losses = []
    for c in self.clients:
        if c.id in benign_clients:
            cl, ns = c.train_metrics()
            num_samples.append(ns)
            losses.append(cl * 1.0)
            
            # Store individual client loss (average per client)
            if ns > 0:
                self.client_losses[c.id] = cl / ns
            else:
                self.client_losses[c.id] = 0.0

    ids = benign_clients

    return ids, num_samples, losses
```

### **What Changed**
- Added 4 lines after the `losses.append()` line
- Stores average loss per client in `self.client_losses[c.id]`
- Includes safety check: only compute average if `ns > 0`
- Falls back to 0.0 if no samples

### **How It Works**
1. `cl` = cumulative loss across all samples for this client
2. `ns` = number of training samples for this client  
3. `cl / ns` = average loss per sample (what we store)
4. This value persists until `save_client_round_data()` writes it to file

---

## **Expected Results**

**Before Fix:**
```
Client Number,Loss Value,Selection Count,Aggregation Count
0,0.0000,10,0
1,0.0000,10,0
2,0.0000,10,0
```

**After Fix (Example):**
```
Client Number,Loss Value,Selection Count,Aggregation Count
0,0.6234,10,0
1,0.5892,10,0
2,0.7145,10,0
```

---

## **Backup Information**

**Backup Location**: `/Users/kasunerandawijethilake/PhD/FED_AA_2/FedAA/Backup/serverbase_BEFORE_LOSS_FIX.py`

**Rollback Command** (if needed):
```bash
cp /Users/kasunerandawijethilake/PhD/FED_AA_2/FedAA/Backup/serverbase_BEFORE_LOSS_FIX.py /Users/kasunerandawijethilake/PhD/FED_AA_2/FedAA/servers/serverbase.py
```

---

## **Testing Instructions**

1. **Run training with the modified code:**
   ```bash
   cd /Users/kasunerandawijethilake/PhD/FED_AA_2/FedAA
   rm -f RESULTS/clients_*.txt
   python main.py
   ```

2. **Check results file after first round:**
   ```bash
   head -5 RESULTS/clients_*.txt
   ```

3. **Expected output:**
   - Loss values should NOT be 0.0000
   - Each client should have a different loss value (e.g., 0.62, 0.59, 0.71, etc.)
   - Selection counts should match actual training rounds

4. **Verify file stays at 100 records:**
   ```bash
   wc -l RESULTS/clients_*.txt
   # Should show 101 (1 header + 100 data rows)
   ```

---

## **Related Code Context**

### **How losses are used:**
1. **Stored in `train_metrics()`** → `self.client_losses[c.id] = cl / ns`
2. **Saved to file in `save_client_round_data()`** → Writes all 100 client records with latest losses
3. **Called from `evaluate()`** → `train_metrics()` is called during each round's evaluation

### **Call Flow:**
```
evaluate() [serverFedAA.py:109]
  ↓
train_metrics() [serverbase.py:232] ← MODIFIED HERE
  ↓
For each client: c.train_metrics()
  ↓
Store: self.client_losses[c.id] = cl / ns ← NEW CODE
  ↓
save_client_round_data() [serverbase.py:290]
  ↓
Write to clients_<Date>.txt with actual loss values
```

---

## **Impact Analysis**

- **Impact**: Low (1 method, 4 lines added)
- **Risk**: None (read-only operation, no side effects)
- **Performance**: No measurable impact (loss division happens once per round per client)
- **Breaking Changes**: None (signature and return values unchanged)

---

## **Verification Checklist**

- [x] Backup created
- [x] Plan documented
- [x] Implementation ready
- [ ] Code applied
- [ ] Training completed
- [ ] Results verified
- [ ] Loss values populated (not 0.0000)

---

## **What `num_sel_clients` is Actually Used For**

Looking at grep results, `num_sel_clients` (=10) is ONLY used in ONE place:

**serverFedAA.py, line 32 (in `get_state()` method):**
```python
random_indices = torch.tensor(random.sample(range(0, self.num_clients), self.num_sel_clients))
self.selected_clients = [self.clients[i] for i in random_indices]
```

**That's it.** The parameter only controls:
- ✅ Which 10 clients to use for computing the state
- ✅ Which 10 clients to consider for aggregation selection

It does **NOT** control:
- ❌ Which clients actually train (all 100 train)
- ❌ Communication patterns
- ❌ Computational efficiency

---

## **The Design Issue**

### **Possible Intended Design (Option A):**
"Train only selected clients" - typical federated learning
- Select 10 clients per round
- Train only those 10
- Aggregate from those 10
- All others remain idle
- **Advantage**: Communication and computation efficient
- **Uses parameter**: `num_sel_clients=10`

### **Actual Current Design (Option B):**
"Train all clients, select for aggregation" - what's happening now
- Train ALL 100 clients every round
- Use 10 selected clients' models to compute RL state
- Aggregate from 5 (selected from the 10)
- **Advantage**: All clients get updated every round
- **Ignores parameter**: `num_sel_clients` is not respected for training

---

## **Summary Table**

| Question | Answer |
|----------|--------|
| **Are all 100 clients trained each round?** | ✅ YES |
| **Does the config say only 10 should train?** | ✅ YES (via `num_sel_clients=10`) |
| **Is there a mismatch?** | ✅ YES |
| **Is `num_sel_clients` used for training selection?** | ❌ NO |
| **Is `num_sel_clients` used for anything?** | ✅ YES - only for state computation |
| **Is this intentional or a bug?** | ⚠️ UNCLEAR |

---

## **The Three Interpretations**

### **1. It's Intentional Design**
"We train all clients but use a subset for RL decision-making"
- Current behavior is correct
- Parameter name is misleading
- Should be called `num_state_clients` instead
- This is actually valid for some FL scenarios

### **2. It's an Incomplete Implementation**
Original code was supposed to:
- Select 10 clients for training
- But developer forgot to filter the training loop
- Currently it trains all 100

### **3. The Design Evolved**
Started as selective training, but evolved to:
- Always train all clients (robustness)
- Use selective sampling for RL optimization only

