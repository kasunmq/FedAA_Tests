# Modification Plan: Keep clients_<Date>.txt at 100 Records Maximum

## Current Behavior (INCORRECT)
```
clients_<Date>.txt after Round 0: 100 rows (1 per client)
clients_<Date>.txt after Round 1: 200 rows (100 + 100 new)
clients_<Date>.txt after Round 2: 300 rows (100 + 100 + 100 new)
```

## Desired Behavior (CORRECT)
```
clients_<Date>.txt after Round 0: 100 rows (1 per client)
clients_<Date>.txt after Round 1: 100 rows (updated records, not new rows)
clients_<Date>.txt after Round 2: 100 rows (updated records, not new rows)
```

---

## Areas to Modify

### 1. **File: `servers/serverbase.py`**
   **Location:** `save_client_round_data()` method (lines 286-293)
   
   **Current Code:**
   ```python
   def save_client_round_data(self, round_num):
       """Save client-level data for the current round"""
       with open(self.clients_results_file, 'a') as f:  # APPENDING - WRONG!
           for client_id in range(self.num_clients):
               loss = self.client_losses.get(client_id, 0.0)
               selection_count = self.client_selection_count.get(client_id, 0)
               aggregation_count = self.client_aggregation_count.get(client_id, 0)
               f.write(f"{client_id},{loss:.4f},{selection_count},{aggregation_count}\n")
   ```
   
   **After Modification (Read-Update-Write approach):**
   ```python
   def save_client_round_data(self, round_num):
       """Save/update client-level data - maintains 100 client records only"""
       # Read existing data if file exists
       client_data = {}
       if os.path.exists(self.clients_results_file):
           try:
               with open(self.clients_results_file, 'r') as f:
                   # Skip header
                   next(f)
                   for line in f:
                       parts = line.strip().split(',')
                       if len(parts) == 4:
                           client_data[int(parts[0])] = line.strip()
           except:
               pass
       
       # Update client records with current round data
       for client_id in range(self.num_clients):
           loss = self.client_losses.get(client_id, 0.0)
           selection_count = self.client_selection_count.get(client_id, 0)
           aggregation_count = self.client_aggregation_count.get(client_id, 0)
           
           # Update or add client record
           client_data[client_id] = f"{client_id},{loss:.4f},{selection_count},{aggregation_count}"
       
       # Write all client records back to file (exactly 100 rows)
       with open(self.clients_results_file, 'w') as f:
           f.write("Client Number,Loss Value,Selection Count,Aggregation Count\n")
           for client_id in range(self.num_clients):
               if client_id in client_data:
                   f.write(client_data[client_id] + "\n")
   ```

---

## Summary of Changes

| File | Method | Change Type | What |
|------|--------|-------------|------|
| serverbase.py | `save_client_round_data()` | REPLACE | Use read-update-write instead of append |

---

## File Output Behavior

### Before Modification (WRONG)
```
Round 0: 101 lines (1 header + 100 data)
Round 1: 201 lines (1 header + 200 data)
Round 2: 301 lines (1 header + 300 data)
Round N: (N+1)*100 + 1 lines
```

### After Modification (CORRECT)
```
Round 0: 101 lines (1 header + 100 data)
Round 1: 101 lines (1 header + 100 data - UPDATED)
Round 2: 101 lines (1 header + 100 data - UPDATED)
Round N: 101 lines (always exactly 100 data rows + 1 header)
```

---

## Example Data

### Before (100 records per round - WRONG)
```
Client Number,Loss Value,Selection Count,Aggregation Count
0,0.2345,1,0
1,0.1876,1,1
...
99,0.1945,1,0
0,0.2110,2,0           ← Round 1 Client 0 (new row)
1,0.1654,2,1           ← Round 1 Client 1 (new row)
...
99,0.1712,2,0          ← Round 1 Client 99 (new row)
```

### After (100 records only - CORRECT)
```
Client Number,Loss Value,Selection Count,Aggregation Count
0,0.2110,2,0           ← Client 0 (updated from Round 0)
1,0.1654,2,1           ← Client 1 (updated from Round 0)
2,0.3012,2,1           ← Client 2 (updated from Round 0)
...
99,0.1712,2,0          ← Client 99 (updated from Round 0)
```

---

## Key Benefits

✅ **File Size:** Stays constant (~2-3 KB instead of growing to 100+ KB)
✅ **Data Clarity:** Always shows current status of each client
✅ **Easy Analysis:** Simple to open in Excel and analyze
✅ **Efficient Storage:** Only 100 rows to maintain and analyze
✅ **Real-time Updates:** Each round updates all 100 client records

---

## How It Works

1. **Round 0 ends:**
   - Read file (doesn't exist yet, skip)
   - Update all 100 client records
   - Write all 100 records to file

2. **Round 1 ends:**
   - Read existing 100 records from file
   - Update all 100 client records with new values
   - Write all 100 records back to file (overwrites old data)

3. **Round N ends:**
   - Read existing 100 records from file
   - Update all 100 client records with new values
   - Write all 100 records back to file (overwrites old data)

---

## Notes

✅ **Safe Implementation:**
- No breaking changes
- Uses read-update-write pattern
- Preserves all client data (updates instead of appends)
- Minimal performance impact

✅ **Data Integrity:**
- Always maintains exactly 100 records
- No duplicate clients
- All counts preserved and updated
- Loss values always current

---

**Ready to implement when you confirm!**
