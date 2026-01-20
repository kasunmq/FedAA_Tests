# Modification Plan: Save Global & Local Results to Separate Files

## Overview
You want to save the evaluation results (printed at end of each round) to two separate files:
- `local_<Date>.txt` - For local model evaluation results
- `global_<Date>.txt` - For global model evaluation results

---

## Areas to Modify

### 1. **File: `servers/serverbase.py`**
   **Location:** `__init__` method (lines ~19-60)
   
   **What to Add:** Create RESULTS folder and initialize file handles for both global and local results files
   
   **Current Code (lines 19-60):**
   ```python
   def __init__(self, args, times):
       # Set up the main attributes
       self.args = args
       self.device = args.device
       # ... more initialization ...
       self.rs_train_loss = []
       self.all_reward = [[] for _ in range(self.global_rounds)]
       self.times = times
       self.replay_buffer = ExperienceReplayBuffer(args.aggre_num, args.aggre_num, max_size=args.buffer_size)
   ```
   
   **After Modification (ADD these lines before closing the `__init__` method):**
   ```python
   # Initialize result files for global and local models
   # Create RESULTS folder if it doesn't exist
   script_dir = os.path.dirname(os.path.abspath(__file__))
   project_dir = os.path.dirname(script_dir)  # Go up one level from servers/
   self.results_dir = os.path.join(project_dir, "RESULTS")
   if not os.path.exists(self.results_dir):
       os.makedirs(self.results_dir)
   
   # Create file names with timestamp
   time_now = datetime.now().strftime("%Y-%m-%d_%H-%M-%S")
   self.global_results_file = os.path.join(self.results_dir, f"global_{time_now}.txt")
   self.local_results_file = os.path.join(self.results_dir, f"local_{time_now}.txt")
   
   # Initialize file headers
   header = "Round No,Averaged Train Loss,Averaged Test Accuracy,Averaged Test AUC,Std Test Accuracy,Std Test AUC\n"
   with open(self.global_results_file, 'w') as f:
       f.write(header)
   with open(self.local_results_file, 'w') as f:
       f.write(header)
   ```

---

### 2. **File: `servers/serverbase.py`**
   **Location:** `evaluate()` method (lines ~216-242)
   
   **What to Modify:** Pass round number and model type to `evaluate()` so it can write to the correct file
   
   **Current Function Signature:**
   ```python
   def evaluate(self, benign_clients, acc=None, loss=None):
   ```
   
   **After Modification (Add parameters):**
   ```python
   def evaluate(self, benign_clients, acc=None, loss=None, round_num=None, model_type='global'):
   ```
   
   **Current Code at end of evaluate() (lines ~237-242):**
   ```python
   print("Averaged Train Loss: {:.4f}".format(train_loss))
   print("Averaged Test Accurancy: {:.4f}".format(test_acc))
   print("Averaged Test AUC: {:.4f}".format(test_auc))
   print("Std Test Accurancy: {:.4f}".format(np.std(accs)))
   print("Std Test AUC: {:.4f}".format(np.std(aucs)))

   return stats, train_loss
   ```
   
   **After Modification (ADD before return statement):**
   ```python
   print("Averaged Train Loss: {:.4f}".format(train_loss))
   print("Averaged Test Accurancy: {:.4f}".format(test_acc))
   print("Averaged Test AUC: {:.4f}".format(test_auc))
   print("Std Test Accurancy: {:.4f}".format(np.std(accs)))
   print("Std Test AUC: {:.4f}".format(np.std(aucs)))
   
   # Save results to appropriate file
   if round_num is not None:
       result_line = f"{round_num},{train_loss:.4f},{test_acc:.4f},{test_auc:.4f},{np.std(accs):.4f},{np.std(aucs):.4f}\n"
       if model_type == 'global':
           with open(self.global_results_file, 'a') as f:
               f.write(result_line)
       elif model_type == 'local':
           with open(self.local_results_file, 'a') as f:
               f.write(result_line)

   return stats, train_loss
   ```

---

### 3. **File: `servers/serverFedAA.py`**
   **Location:** `train()` method (lines ~68-129)
   
   **What to Modify:** Update the two `evaluate()` calls to include round number and model type
   
   **Current Code (Line ~68):**
   ```python
   print("\nEvaluate global model")
   
   action = self.agent.select_action(state)
   self.receive_models_RL(action, self.aggre_clients)
   self.aggregate_parameters()
   self.send_models()
   self.evaluate(self.benign_clients)
   ```
   
   **After Modification:**
   ```python
   print("\nEvaluate global model")
   
   action = self.agent.select_action(state)
   self.receive_models_RL(action, self.aggre_clients)
   self.aggregate_parameters()
   self.send_models()
   self.evaluate(self.benign_clients, round_num=i, model_type='global')
   ```

   **Current Code (Line ~103):**
   ```python
   print("\nEvaluate local model")
   self.evaluate(self.benign_clients)
   ```
   
   **After Modification:**
   ```python
   print("\nEvaluate local model")
   self.evaluate(self.benign_clients, round_num=i, model_type='local')
   ```

---

## Summary of Changes

| File | Method | Change Type | Lines | What |
|------|--------|-------------|-------|------|
| serverbase.py | `__init__()` | ADD | ~60 | Initialize RESULTS folder and files |
| serverbase.py | `evaluate()` | MODIFY | Function signature | Add `round_num` and `model_type` parameters |
| serverbase.py | `evaluate()` | ADD | ~242 | Write results to files before return |
| serverFedAA.py | `train()` | MODIFY | ~68 | Pass parameters to evaluate() call |
| serverFedAA.py | `train()` | MODIFY | ~103 | Pass parameters to evaluate() call |

---

## File Output Structure

After running, you'll get:
```
FedAA/
├── RESULTS/
│   ├── global_2026-01-21_15-30-45.txt
│   └── local_2026-01-21_15-30-45.txt
├── servers/
├── clients/
└── main.py
```

**File Format (both global and local):**
```
Round No,Averaged Train Loss,Averaged Test Accuracy,Averaged Test AUC,Std Test Accuracy,Std Test AUC
0,1.9780,0.4117,0.7584,0.2616,0.1407
1,1.8650,0.4225,0.7642,0.2580,0.1398
2,1.7540,0.4350,0.7698,0.2550,0.1385
...
```

---

## Notes

✅ **What stays the same:**
- All console print statements remain unchanged
- All existing functionality preserved
- No logic changes, only data recording

✅ **What's new:**
- Creates a `RESULTS/` folder automatically
- Creates two separate files with timestamp
- Appends results to files after each round evaluation
- Files use CSV format for easy analysis

✅ **Safe to implement:**
- No breaking changes
- Uses existing data (just saves it)
- Datetime import already exists
- OS module already imported

---

**Ready to implement when you confirm!**
