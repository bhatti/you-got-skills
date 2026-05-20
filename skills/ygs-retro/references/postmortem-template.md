# Post-Mortem Template

Use when `/ygs-retro` identifies incidents worth documenting.

## Structure

### 1. Title and Summary
- Customer-facing problem statement
- Duration and timeline (start → detection → resolution)
- Quantified impact (users affected, revenue lost, SLA breached)
- High-level cause (one sentence)
- Resolution method

### 2. Timeline
- Start from initial trigger (not notification)
- Consistent timezone throughout
- Bold key milestones
- Note: time to detect, time to diagnose, time to mitigate

### 3. Impact
- Specific quantification ("4,200 users received errors for 23 minutes")
- NOT vague statements ("users experienced issues")

### 4. Root Cause Analysis (Five Whys)
1. Why did the system fail? → [answer]
2. Why was [answer] possible? → [deeper answer]
3. Why wasn't [deeper answer] prevented? → [systemic issue]
4. Why wasn't it detected sooner? → [monitoring gap]
5. Why wasn't impact contained? → [isolation gap]

### 5. Lessons Learned
- Numbered, universal principles (applicable beyond this incident)
- Not "we should have been more careful" — systemic fixes only

### 6. Action Items
For each action:
- [ ] Specific fix (not "investigate better monitoring")
- **Owner:** [name]
- **Deadline:** [date]
- **Priority:** P0/P1/P2
- **Prevents:** Links to which "Why" it addresses

## Anti-Patterns to Avoid
- Stopping at "human error" without systemic analysis
- Vague action items ("look into it")
- No owner or deadline on actions
- Writing for insiders only (assume new team member reads this)
- Same root cause appearing across multiple post-mortems (track this!)

## Swiss Cheese Model
Multiple layers failed simultaneously. Identify ALL layers:
- Could code have prevented it?
- Could tests have caught it?
- Could monitoring have detected it faster?
- Could deployment process have limited blast radius?
- Could documentation have helped responders?
