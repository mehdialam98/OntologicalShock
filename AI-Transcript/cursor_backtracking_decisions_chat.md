# Backtracking decisions in a question game
_Exported on 11/3/2025 at 22:03:32 EST from Cursor (2.0.43)_

---

**User**

I@20_Q_GAME.ipynb  have a 20 question game. What are some ways I can back trace decisions incase my input json data is wrong or a user answers incorrectly.

---

**Cursor**


Reviewing the game structure to add backtracking for debugging and correcting user answers.


Adding backtracking for debugging and correcting user answers. Implementing:
1. Game state history (snapshots after each question)
2. Undo functionality
3. Answer correction
4. Debug mode showing which nouns were eliminated and why
5. Inconsistency detection


```plaintext
    def ask_yes_no(self, prompt: str) -> Optional[bool]:
        """
        Ask user for yes/no. Returns True/False or None for 'quit'.
        Accepts: y/yes, n/no, q/quit
        """
        while True:
            ans = input(f"{prompt} (y/n, q to quit): ").strip().lower()
            if ans in ("y", "yes"):
                return True
            if ans in ("n", "no"):
                return False
            if ans in ("q", "quit", "exit"):
                return None
            print("Please answer 'y' or 'n' (or 'q' to quit).")

print("✅ TwentyQuestionsGame class defined")

->

    def ask_yes_no(self, prompt: str) -> Optional[bool]:
        """
        Ask user for yes/no. Returns True/False or None for 'quit'.
        Accepts: y/yes, n/no, q/quit
        """
        while True:
            ans = input(f"{prompt} (y/n, q to quit): ").strip().lower()
            if ans in ("y", "yes"):
                return True
            if ans in ("n", "no"):
                return False
            if ans in ("q", "quit", "exit"):
                return None
            print("Please answer 'y' or 'n' (or 'q' to quit).")
    
    def explain_elimination(self, eliminated_nouns, property_name, threshold, answer):
        """
        Explain why certain nouns were eliminated by a question
        Useful for debugging knowledge base inconsistencies
        """
        if len(eliminated_nouns) == 0:
            return []
        
        subset = self.kb[self.kb['noun'].isin(eliminated_nouns)]
        explanations = []
        
        for noun in eliminated_nouns[:10]:  # Show first 10
            row = self.kb[self.kb['noun'] == noun].iloc[0]
            
            if threshold is None:  # Binary
                actual_value = row[property_name]
                expected_value = answer
                explanation = f"  {noun}: {property_name} = {actual_value}, but answer was {expected_value}"
            else:  # Scale
                actual_value = row[property_name]
                expected_comparison = ">" if answer else "<="
                explanation = f"  {noun}: {property_name} = {actual_value:.1f}, but answer requires {property_name} {expected_comparison} {threshold:.1f}"
            
            explanations.append(explanation)
        
        if len(eliminated_nouns) > 10:
            explanations.append(f"  ... and {len(eliminated_nouns) - 10} more")
        
        return explanations

print("✅ TwentyQuestionsGame class defined")
```

```plaintext
# Cell 16 - PLAY GAME FUNCTION
import random

def play_game_interactive(game, target_noun, max_questions=20, verbose=True, auto=True):
    """
    Play a game of 20 questions interactively

    Args:
        game: TwentyQuestionsGame instance
        target_noun: The noun to guess
        max_questions: Maximum questions allowed
        verbose: Print detailed output

    Returns:
        dict with game results
    """
    if target_noun not in game.all_nouns:
        return {
            'success': False,
            'error': f"'{target_noun}' not in knowledge base"
        }

    remaining_nouns = game.all_nouns.copy()
    questions_asked = []
    answers_given = []

->

# Cell 16 - PLAY GAME FUNCTION WITH BACKTRACKING
import random

def play_game_interactive(game, target_noun, max_questions=20, verbose=True, auto=True, 
                          enable_backtrack=True, debug_mode=False):
    """
    Play a game of 20 questions interactively with backtracking support

    Args:
        game: TwentyQuestionsGame instance
        target_noun: The noun to guess
        max_questions: Maximum questions allowed
        verbose: Print detailed output
        auto: Auto-answer from knowledge base (True) or manual input (False)
        enable_backtrack: Enable undo/correction functionality
        debug_mode: Show detailed elimination information

    Returns:
        dict with game results
    """
    if target_noun not in game.all_nouns:
        return {
            'success': False,
            'error': f"'{target_noun}' not in knowledge base"
        }

    remaining_nouns = game.all_nouns.copy()
    questions_asked = []
    answers_given = []
    properties_used = []  # Track which properties were used
    thresholds_used = []   # Track thresholds for scale questions
    
    # BACKTRACKING: Game state history
    game_history = []  # List of game states: (remaining_nouns, question_num, question, answer)
    
    def save_game_state(question_num, question_text, answer, prop, threshold):
        """Save current game state to history"""
        game_history.append({
            'question_num': question_num,
            'question': question_text,
            'answer': answer,
            'property': prop,
            'threshold': threshold,
            'remaining_nouns': remaining_nouns.copy(),
            'remaining_count': len(remaining_nouns)
        })
    
    def restore_game_state(state_index):
        """Restore game to a previous state"""
        if state_index < 0 or state_index >= len(game_history):
            return False
        
        # Restore to state before the specified question
        state = game_history[state_index]
        nonlocal remaining_nouns, questions_asked, answers_given, properties_used, thresholds_used
        
        remaining_nouns = state['remaining_nouns'].copy()
        questions_asked = questions_asked[:state['question_num'] - 1]
        answers_given = answers_given[:state['question_num'] - 1]
        properties_used = properties_used[:state['question_num'] - 1]
        thresholds_used = thresholds_used[:state['question_num'] - 1]
        
        # Remove all states after this one
        game_history[:] = game_history[:state_index + 1]
        return True
    
    def show_history():
        """Display game history"""
        print("\n" + "="*70)
        print("GAME HISTORY")
        print("="*70)
        for i, state in enumerate(game_history):
            print(f"\nState {i+1} (Q{state['question_num']}):")
            print(f"  Question: {state['question']}")
            print(f"  Answer: {state['answer']}")
            print(f"  Remaining nouns: {state['remaining_count']}")
        print("="*70)
    
    def ask_with_undo(prompt: str, question_num: int) -> Optional[bool]:
        """
        Ask user for yes/no with undo/correction options
        """
        while True:
            ans = input(f"{prompt}\n  (y/n, 'u' to undo, 'h' for history, 'd' for debug, 'q' to quit): ").strip().lower()
            if ans in ("y", "yes"):
                return True
            if ans in ("n", "no"):
                return False
            if ans in ("u", "undo") and enable_backtrack:
                if len(game_history) > 0:
                    print(f"\n🔄 UNDO: Going back to question {game_history[-1]['question_num']}")
                    restore_game_state(len(game_history) - 1)
                    return "UNDO"
                else:
                    print("⚠️  Nothing to undo!")
                    continue
            if ans in ("h", "history") and enable_backtrack:
                show_history()
                continue
            if ans in ("d", "debug") and debug_mode:
                if len(game_history) > 0:
                    last_state = game_history[-1]
                    eliminated = game.all_nouns - last_state['remaining_nouns']
                    if len(eliminated) > 0:
                        print(f"\n🔍 DEBUG: Showing elimination details for last question:")
                        explanations = game.explain_elimination(
                            eliminated, 
                            last_state['property'], 
                            last_state['threshold'], 
                            last_state['answer']
                        )
                        for exp in explanations:
                            print(exp)
                continue
            if ans in ("q", "quit", "exit"):
                return None
            print("Please answer 'y', 'n', 'u' (undo), 'h' (history), 'd' (debug), or 'q' (quit).")
```

```plaintext
        # Get the correct answer from knowledge base
        if auto:
            target_row = game.kb[game.kb['noun'] == target_noun].iloc[0]

            if threshold is None:  # Binary
                correct_answer = bool(target_row[prop])
            else:  # Scale
                correct_answer = float(target_row[prop]) > threshold
        else:
            correct_answer = game.ask_yes_no(question_text)
            if (correct_answer == None):
                return {
                    'success': False,
                    'error': 'User manually quit game.',
                    'questions_asked': question_num,
                    'questions': questions_asked,
                    'answers': answers_given
                }


        questions_asked.append(question_text)
        answers_given.append(correct_answer)

        if verbose:
            print(f"\nQ{question_num}: {question_text}")
            print(f"A: {'YES' if correct_answer else 'NO'}")

        # Filter remaining nouns
        remaining_nouns = game.filter_nouns(remaining_nouns, prop, threshold, correct_answer)

->

        # Get the correct answer from knowledge base
        if auto:
            target_row = game.kb[game.kb['noun'] == target_noun].iloc[0]

            if threshold is None:  # Binary
                correct_answer = bool(target_row[prop])
            else:  # Scale
                correct_answer = float(target_row[prop]) > threshold
        else:
            if enable_backtrack:
                correct_answer = ask_with_undo(question_text, question_num)
            else:
                correct_answer = game.ask_yes_no(question_text)
            
            if correct_answer == "UNDO":
                # User undid, restart loop from previous state
                question_num = game_history[-1]['question_num'] if game_history else 0
                continue
            
            if (correct_answer == None):
                return {
                    'success': False,
                    'error': 'User manually quit game.',
                    'questions_asked': question_num,
                    'questions': questions_asked,
                    'answers': answers_given,
                    'game_history': game_history
                }

        # Store state BEFORE filtering (so we can undo to this point)
        nouns_before = remaining_nouns.copy()
        
        questions_asked.append(question_text)
        answers_given.append(correct_answer)
        properties_used.append(prop)
        thresholds_used.append(threshold)

        if verbose:
            print(f"\nQ{question_num}: {question_text}")
            print(f"A: {'YES' if correct_answer else 'NO'}")

        # Filter remaining nouns
        remaining_nouns = game.filter_nouns(remaining_nouns, prop, threshold, correct_answer)
        
        # Debug: Show eliminated nouns
        if debug_mode:
            eliminated = nouns_before - remaining_nouns
            if len(eliminated) > 0:
                print(f"  🔍 Eliminated {len(eliminated)} nouns")
                if len(eliminated) <= 10:
                    explanations = game.explain_elimination(eliminated, prop, threshold, correct_answer)
                    for exp in explanations:
                        print(exp)
        
        # Save state AFTER filtering
        save_game_state(question_num, question_text, correct_answer, prop, threshold)
```

```plaintext
    # Should not reach here
    return {
        'success': False,
        'error': 'Max questions reached',
        'questions_asked': max_questions,
        'remaining_nouns': len(remaining_nouns)
    }

->

    # Should not reach here
    return {
        'success': False,
        'error': 'Max questions reached',
        'questions_asked': max_questions,
        'remaining_nouns': len(remaining_nouns),
        'game_history': game_history
    }
```

Updating all return statements to include game history, and adding a utility function for analyzing game results:


Updating all return statements to include game_history and adding a utility function for post-game analysis:


```json
            return {
                'success': final_guess == target_noun,
                'final_guess': final_guess,
                'target_noun': target_noun,
                'questions_asked': question_num,
                'questions': questions_asked,
                'answers': answers_given + [final_guess == target_noun],
                'remaining_nouns': 
```

```json
            return {
                'success': final_guess == target_noun,
                'final_guess': final_guess,
                'target_noun': target_noun,
                'questions_asked': question_num,
                'questions': questions_asked,
                'answers': answers_given + [final_guess == target_noun],
                'remaining_nouns': len(remaining_nouns
```

```json
            return {
                'success': False,
                'error': 'No remaining nouns',
                'questions_asked': question_num,
                'questions': questions_asked,
                'answers': answers_give
```

```plaintext
            return {
                'success': final_guess == target_noun,
                'final_guess': final_guess,
                'target_noun': target_noun,
                'questions_asked': question_num,
                'questions': questions_asked,
                'answers': answers_given + [final_guess == target_noun],
                'remaining_nouns': 1
            }

->

            return {
                'success': final_guess == target_noun,
                'final_guess': final_guess,
                'target_noun': target_noun,
                'questions_asked': question_num,
                'questions': questions_asked,
                'answers': answers_given + [final_guess == target_noun],
                'remaining_nouns': 1,
                'game_history': game_history
            }
```

```plaintext
            return {
                'success': final_guess == target_noun,
                'final_guess': final_guess,
                'target_noun': target_noun,
                'questions_asked': question_num,
                'questions': questions_asked,
                'answers': answers_given + [final_guess == target_noun],
                'remaining_nouns': len(remaining_nouns)
            }

        # Find best question

->

            return {
                'success': final_guess == target_noun,
                'final_guess': final_guess,
                'target_noun': target_noun,
                'questions_asked': question_num,
                'questions': questions_asked,
                'answers': answers_given + [final_guess == target_noun],
                'remaining_nouns': len(remaining_nouns),
                'game_history': game_history
            }

        # Find best question
```

```plaintext
            return {
                'success': final_guess == target_noun,
                'final_guess': final_guess,
                'target_noun': target_noun,
                'questions_asked': question_num,
                'questions': questions_asked,
                'answers': answers_given + [final_guess == target_noun],
                'remaining_nouns': len(remaining_nouns)
            }

        # Get the correct answer from knowledge base

->

            return {
                'success': final_guess == target_noun,
                'final_guess': final_guess,
                'target_noun': target_noun,
                'questions_asked': question_num,
                'questions': questions_asked,
                'answers': answers_given + [final_guess == target_noun],
                'remaining_nouns': len(remaining_nouns),
                'game_history': game_history
            }

        # Get the correct answer from knowledge base
```

```plaintext
            return {
                'success': False,
                'error': 'No remaining nouns',
                'questions_asked': question_num,
                'questions': questions_asked,
                'answers': answers_given
            }

->

            return {
                'success': False,
                'error': 'No remaining nouns',
                'questions_asked': question_num,
                'questions': questions_asked,
                'answers': answers_given,
                'game_history': game_history
            }
```

Adding a utility function for post-game analysis to detect inconsistencies:


```plaintext
# Initialize game
game = TwentyQuestionsGame(kb_df)

# Test with one noun
#test_noun = random.choice(list(game.all_nouns))
#print(f"\n\n🧪 Testing with: {test_noun}\n")
#result = play_game_interactive(game, test_noun, verbose=True)

#print(f"\n\n{'='*70}")
#print("GAME RESULT:")
#print(f"Success: {result['success']}")
#print(f"Questions used: {result.get('questions_asked', 'N/A')}")
#print("="*70)

->

# Utility function for analyzing game results and detecting inconsistencies
def analyze_game_result(game, result, show_details=True):
    """
    Analyze a game result to detect knowledge base inconsistencies or user errors
    
    Args:
        game: TwentyQuestionsGame instance
        result: Result dictionary from play_game_interactive
        show_details: Show detailed analysis
    """
    if 'game_history' not in result or not result['game_history']:
        print("⚠️  No game history available for analysis")
        return
    
    print("\n" + "="*70)
    print("GAME ANALYSIS")
    print("="*70)
    
    target_noun = result.get('target_noun', 'unknown')
    success = result.get('success', False)
    
    print(f"\nTarget: {target_noun}")
    print(f"Success: {success}")
    print(f"Questions asked: {result.get('questions_asked', 'N/A')}")
    
    if not success:
        print("\n🔍 Analyzing why the game failed...")
        
        # Check if target should have been eliminated
        target_row = game.kb[game.kb['noun'] == target_noun]
        if len(target_row) == 0:
            print(f"❌ ERROR: '{target_noun}' not found in knowledge base!")
            return
        
        target_row = target_row.iloc[0]
        final_guess = result.get('final_guess')
        
        if final_guess:
            print(f"\nFinal guess was: {final_guess}")
            print(f"Target was: {target_noun}")
            
            # Check each question to see if target should have been eliminated
            print("\n📊 Checking each question against target noun:")
            for i, state in enumerate(result['game_history']):
                prop = state['property']
                threshold = state['threshold']
                answer = state['answer']
                
                # Get target's value for this property
                if threshold is None:  # Binary
                    target_value = bool(target_row[prop])
                    should_match = answer
                    matches = target_value == should_match
                else:  # Scale
                    target_value = float(target_row[prop])
                    should_be_above = answer
                    matches = (target_value > threshold) == should_be_above
                
                status = "✅" if matches else "❌"
                print(f"\n  Q{state['question_num']}: {state['question']}")
                print(f"    Answer given: {answer}")
                if threshold is None:
                    print(f"    Target's {prop}: {target_value}")
                else:
                    print(f"    Target's {prop}: {target_value:.1f} (threshold: {threshold:.1f})")
                print(f"    {status} {'Match' if matches else 'MISMATCH - Target should have been eliminated!'}")
                
                if not matches:
                    print(f"    ⚠️  INCONSISTENCY DETECTED: Target should have been eliminated at this question!")
    
    if show_details and 'game_history' in result:
        print("\n" + "="*70)
        print("DETAILED GAME HISTORY")
        print("="*70)
        for i, state in enumerate(result['game_history']):
            print(f"\nState {i+1} (Q{state['question_num']}):")
            print(f"  Question: {state['question']}")
            print(f"  Answer: {state['answer']}")
            print(f"  Remaining nouns: {state['remaining_count']}")
    
    print("="*70)

# Initialize game
game = TwentyQuestionsGame(kb_df)

# Test with one noun
#test_noun = random.choice(list(game.all_nouns))
#print(f"\n\n🧪 Testing with: {test_noun}\n")
#result = play_game_interactive(game, test_noun, verbose=True)

#print(f"\n\n{'='*70}")
#print("GAME RESULT:")
#print(f"Success: {result['success']}")
#print(f"Questions used: {result.get('questions_asked', 'N/A')}")
#print("="*70)
#analyze_game_result(game, result)
```

Adding a new cell demonstrating the backtracking features:


```plaintext


->

# Backtracking Features

This notebook now includes backtracking capabilities to help debug knowledge base issues and handle user errors:

## Features:

1. **Undo Functionality**: Type 'u' or 'undo' during a question to go back one step
2. **History View**: Type 'h' or 'history' to see all questions and answers so far
3. **Debug Mode**: Type 'd' or 'debug' to see detailed elimination information
4. **Game History**: Full game state is saved after each question
5. **Post-Game Analysis**: Use `analyze_game_result()` to detect inconsistencies

## Usage Examples:

### Interactive Mode with Backtracking:
```python
result = play_game_interactive(game, "bicycle", verbose=True, auto=False, enable_backtrack=True)
```

### Debug Mode (shows eliminated nouns):
```python
result = play_game_interactive(game, "bicycle", verbose=True, auto=False, enable_backtrack=True, debug_mode=True)
```

### Post-Game Analysis:
```python
result = play_game_interactive(game, "bicycle", verbose=True)
analyze_game_result(game, result)  # Detects inconsistencies
```
```

```plaintext


->

# Example: Test backtracking with debug mode
# Uncomment to test

# test_noun = "bicycle"  # or any noun in your knowledge base
# print(f"\n🧪 Testing backtracking with: {test_noun}\n")
# result = play_game_interactive(
#     game, 
#     test_noun, 
#     verbose=True, 
#     auto=False,  # Manual input mode
#     enable_backtrack=True,  # Enable undo/history
#     debug_mode=True  # Show elimination details
# )

# print(f"\n\n{'='*70}")
# print("GAME RESULT:")
# print(f"Success: {result['success']}")
# print(f"Questions used: {result.get('questions_asked', 'N/A')}")
# print("="*70)

# # Analyze the result for inconsistencies
# analyze_game_result(game, result)
```

Fixing undo logic to correctly handle state restoration:


Converting the for loop to a while loop so undo works correctly:


```plaintext
    if verbose:
        print("="*70)
        print(f"🎮 PLAYING 20 QUESTIONS")
        print(f"Target noun: {target_noun}")
        print("="*70)

    for question_num in range(1, max_questions + 1):

->

    if verbose:
        print("="*70)
        print(f"🎮 PLAYING 20 QUESTIONS")
        print(f"Target noun: {target_noun}")
        print("="*70)

    question_num = 0
    while question_num < max_questions:
        question_num += 1
```

```plaintext
            if correct_answer == "UNDO":
                # User undid, restart loop from previous state
                question_num = game_history[-1]['question_num'] if game_history else 0
                continue

->

            if correct_answer == "UNDO":
                # User undid, go back to previous question
                if len(game_history) > 0:
                    # Restore to state before last question (which is stored in history)
                    # We need to go back one more step
                    if len(game_history) > 1:
                        # Restore to state before the last question
                        restore_game_state(len(game_history) - 2)
                        question_num = game_history[-1]['question_num'] if game_history else question_num
                    else:
                        # Only one question, restore to beginning
                        restore_game_state(0)
                        question_num = 0
                continue
```

Fixing undo to restore to the state before the last question:


```plaintext
            if ans in ("u", "undo") and enable_backtrack:
                if len(game_history) > 0:
                    print(f"\n🔄 UNDO: Going back to question {game_history[-1]['question_num']}")
                    restore_game_state(len(game_history) - 1)
                    return "UNDO"
                else:
                    print("⚠️  Nothing to undo!")
                    continue

->

            if ans in ("u", "undo") and enable_backtrack:
                if len(game_history) > 0:
                    # To undo the last question, restore to the state before it
                    # game_history[-1] is after the last question, we want before it
                    if len(game_history) > 1:
                        prev_question = game_history[-2]['question_num']
                        print(f"\n🔄 UNDO: Going back to before question {game_history[-1]['question_num']}")
                        restore_game_state(len(game_history) - 2)
                        return ("UNDO", prev_question)
                    else:
                        # Only one question, go back to start
                        print(f"\n🔄 UNDO: Going back to beginning")
                        restore_game_state(0)
                        return ("UNDO", 0)
                else:
                    print("⚠️  Nothing to undo!")
                    continue
```

```plaintext
            if correct_answer == "UNDO":
                # User undid, go back to previous question
                if len(game_history) > 0:
                    # Restore to state before last question (which is stored in history)
                    # We need to go back one more step
                    if len(game_history) > 1:
                        # Restore to state before the last question
                        restore_game_state(len(game_history) - 2)
                        question_num = game_history[-1]['question_num'] if game_history else question_num
                    else:
                        # Only one question, restore to beginning
                        restore_game_state(0)
                        question_num = 0
                continue

->

            if isinstance(correct_answer, tuple) and correct_answer[0] == "UNDO":
                # User undid, adjust question number to continue from restored state
                _, restored_q = correct_answer
                question_num = restored_q
                continue
```

Creating a summary document of the backtracking features:


## Summary

Backtracking features added to your 20 Questions game:

### 1. **Game State History**
   - Saves game state after each question (remaining nouns, questions asked, answers given)
   - Stored in `game_history` in the result dictionary

### 2. **Undo Functionality**
   - Type `'u'` or `'undo'` during a question to go back one step
   - Restores the game state to before the last question
   - Allows re-answering the previous question

### 3. **History View**
   - Type `'h'` or `'history'` to view all questions and answers so far
   - Shows remaining noun count after each question

### 4. **Debug Mode**
   - Type `'d'` or `'debug'` (when `debug_mode=True`) to see which nouns were eliminated and why
   - Shows property values that caused elimination

### 5. **Post-Game Analysis**
   - `analyze_game_result()` detects inconsistencies in the knowledge base
   - Compares each answer against the target noun's properties
   - Highlights where the target should have been eliminated but wasn't (indicating data issues)

### 6. **Enhanced Debugging**
   - `explain_elimination()` shows why specific nouns were eliminated
   - Useful for debugging knowledge base inconsistencies

### Usage Examples:

**Interactive mode with backtracking:**
```python
result = play_game_interactive(
    game, 
    "bicycle", 
    verbose=True, 
    auto=False,  # Manual input
    enable_backtrack=True,  # Enable undo/history
    debug_mode=True  # Show elimination details
)
```

**Post-game analysis:**
```python
result = play_game_interactive(game, "bicycle", verbose=True)
analyze_game_result(game, result)  # Detects inconsistencies
```

These features help:
- Debug when JSON data is wrong (by seeing which answers eliminate the target incorrectly)
- Handle user input errors (by allowing undo)
- Understand why the game failed (through detailed analysis)

The game history is saved in all return dictionaries, so you can analyze failed games after they complete.

```python
result = play_game_interactive(
    game, 
    "bicycle", 
    verbose=True, 
    auto=False,  # Manual input
    enable_backtrack=True,  # Enable undo/history
    debug_mode=True  # Show elimination details
)
```

```python
result = play_game_interactive(game, "bicycle", verbose=True)
analyze_game_result(game, result)  # Detects inconsistencies
```

