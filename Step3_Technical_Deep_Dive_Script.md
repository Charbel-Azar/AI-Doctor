# Step 3: Technical Deep Dive - Allocation Factors Training Script
## Duration: 10-12 minutes

---

## **SLIDE 1: Technical Deep Dive Title**

**[SPEAKER NOTES]**

Alright everyone, now we're going to get our hands dirty and look under the hood. This is where we see exactly how allocation factors work in the code. Don't worry if you're not a SQL expert - I'll walk you through the key concepts step by step.

By the end of this section, you'll understand:
- How the system calculates allocation weights
- What tables are involved and why
- The exact flow from raw costs to allocated costs
- And most importantly - where things can go wrong

Let's dive in.

---

## **SLIDE 2: The Core Concept - Weight-Based Distribution**

**[SPEAKER NOTES]**

At its heart, an allocation factor is just a fancy way of splitting up a pie. 

Here's the fundamental formula that drives everything:

**Weight = AllocationFactorValue / AllocationFactorTotal**

Let me give you a real example. Say we have $100,000 in salary costs for the Litigation department, and we need to split it among three partners:

- Sarah has an ALLTK value of 2.0 (she's a Partner)
- John has 1.5 (he's an Associate)
- Maria has 2.0 (also a Partner)

The total is 5.5.

So Sarah gets: 2.0 / 5.5 = 36.4% → $36,400
John gets: 1.5 / 5.5 = 27.3% → $27,300
Maria gets: 2.0 / 5.5 = 36.4% → $36,400

That weight - that percentage - is what drives every single allocation in the profit module.

---

## **SLIDE 3: Key Data Structures**

**[SPEAKER NOTES]**

Now let's talk about where all this lives in the database. There are four critical tables you need to know:

**First: bi.ProfitHr**
This is your HR master table. It stores:
- One row per person per period
- FTE, headcount, standard hours
- And here's the key: AllocationFactor0 through AllocationFactor9
- That's right - we have 10 slots for different allocation methods

Think of AllocationFactor0 as "ALLTK" - our all-timekeeper allocation. AllocationFactor1 might be "BILLABLE" - allocating based on billable hours only. You can define up to 10 different schemes.

**Second: bi.ProfitHrTemp**
This is our calculation workhorse. The process:
1. Takes ProfitHr
2. Unpivots those 10 allocation factors
3. Calculates totals at different organizational levels
4. Computes the weights

It's a temporary table that gets rebuilt every time we run the profit process.

**Third: bi.ProfitCostRaw**
This is where costs land BEFORE allocation. Costs come in at different granularities:
- Some at the timekeeper level (TK) - like imported salary data
- Some at office+department (OD) - like most GL costs
- Some firm-wide (FW) - like insurance costs

They sit here waiting to be allocated.

**Fourth: bi.ProfitCostWa**
WA stands for Working Attorney. This is the destination - costs AFTER allocation. Every row here is at the timekeeper level, ready to be pushed down to matters.

Think of it as: **Raw → (Allocation Engine) → WA**

---

## **SLIDE 4: The Allocation Flow - Step 1: Define Values**

**[SPEAKER NOTES]**

Let me show you how this actually works in code. We'll follow the ALLTK allocation factor through the entire process.

**Step 1: Define Allocation Factor Values**

*[CLICK to show code snippet]*

Look at this code around line 380 in ProfitProcess. This is where we calculate AllocationFactor0 - the ALLTK factor.

The business rules are:
- Start with FTE (must be non-zero)
- Multiply by title weight:
  - Partners and Counsel: 2.0
  - Associates: 1.5
  - Solicitors: 1.0
  - Lawyers, Law Clerks, Consultants: 0.5
  
But wait - there are MORE filters:
- Must have at least 100 annualized billable hours
- Must be in a practicing office (not Support)
- Must be in a practicing department (not Support)
- Must be in a practicing team (not Support)

Why so many filters? Because we only want to allocate costs to people who are actually generating revenue. If you're a full-time admin partner with zero billable hours, you don't get allocated costs under ALLTK.

So a full-time Partner with 150 billable hours gets: 1.0 FTE × 2.0 = 2.0
A half-time Associate with 120 billable hours gets: 0.5 FTE × 1.5 = 0.75

Those values go into the AllocationFactor0 column of bi.ProfitHr.

---

## **SLIDE 5: The Allocation Flow - Step 2: Calculate Totals**

**[SPEAKER NOTES]**

**Step 2: Calculate Totals at Multiple Granularities**

*[CLICK to show code snippet]*

Now here's where it gets interesting. Look at this code starting around line 765.

We insert into ProfitHrTemp and use an UNPIVOT operation. What's happening?

The system calculates allocation factor TOTALS at different organizational levels:

- **TK** (Timekeeper): Total for just that person (always equals their own value)
- **OD** (Office + Department): Total for everyone in that office+department combo
- **O** (Office): Total for everyone in that office
- **D** (Department): Total for everyone in that department
- **FW** (Firm-Wide): Total for everyone in the entire firm

Why do we need all these levels? Because costs come in at different levels!

For example:
- **Partner compensation** might be tracked at the TK level (we know exactly whose comp it is)
- **Department administrative costs** come in at the OD level (NY Litigation admin costs)
- **Building rent** comes in at the O level (NY Office rent)
- **Firm-wide insurance** comes in at FW level

The system needs to be able to allocate at whatever level the cost naturally exists.

And for each level, we calculate the weight:
**Weight = AllocationFactorValue / AllocationFactorTotal**

So if the NY Litigation department has 5 people with ALLTK values totaling 8.5, and Sarah has 2.0, her weight for OD allocations is 2.0/8.5 = 23.5%.

All of this goes into ProfitHrTemp - one row per person per granularity.

---

## **SLIDE 6: The Allocation Flow - Step 3: Load Raw Costs**

**[SPEAKER NOTES]**

**Step 3: Load Raw Costs at Source Granularity**

*[CLICK to show code snippet]*

Now let's see where costs come from. This is around line 950 - loading GL costs at the Office+Department level.

Notice the key fields:
- **ProfitAccountId**: 'COMP-SALARY.0' (compensation-salary account)
- **SourceId**: 'GL-OD' (tells us this came from GL at OD granularity)
- **AllocationFactor**: 'ALLTK' (we'll use the ALLTK weights to allocate)
- **Granularity**: 'OD' (this is at office+department level)
- **CostWorked**: The actual dollar amount

The system is smart enough to:
1. Read from ProfitGl (where GL data lands)
2. Determine what granularity the cost exists at
3. Know which allocation factor to use (based on ProfitAccountVariant rules)
4. Stage it in ProfitCostRaw

At this point, nothing is allocated yet. We just have costs sitting at various organizational levels, tagged with which allocation factor should be used.

There are multiple INSERT statements like this for:
- **IMP-TK**: Imported costs at TK level
- **GL-TK**: GL costs at TK level
- **GL-O**: GL costs at Office level
- **GL-OD**: GL costs at Office+Department level
- **GL-FW**: GL costs Firm-Wide level

Each one loads costs at the appropriate granularity.

---

## **SLIDE 7: The Allocation Flow - Step 4: Allocate to TKs**

**[SPEAKER NOTES]**

**Step 4: The Magic - Allocate to Timekeepers**

*[CLICK to show code snippet]*

Here it is - the moment everything comes together. This is around line 1230.

Look at this INSERT INTO bi.ProfitCostWa. This is where allocation actually happens.

The key is this join:
```sql
FROM bi.ProfitCostRaw Q
INNER JOIN bi.ProfitHrTemp H ON 
    H.PeriodId = Q.PeriodId
    AND H.AllocationFactor = Q.AllocationFactor
    AND H.Granularity = Q.Granularity
```

We're matching:
- Same period ✓
- Same allocation factor (e.g., ALLTK) ✓
- Same granularity (e.g., OD) ✓

And then there's this beautiful matching logic:
```sql
AND (
    H.Granularity = 'TK'  AND H.WorkingAttorneyId = Q.WorkingAttorneyId OR
    H.Granularity = 'OD'  AND H.OfficeId = Q.OfficeId 
                          AND H.DepartmentId = Q.DepartmentId OR
    H.Granularity = 'O'   AND H.OfficeId = Q.OfficeId OR
    H.Granularity = 'FW'  -- No filter needed
)
```

This says:
- If it's TK-level cost, match on the exact person
- If it's OD-level cost, match everyone in that office+department
- If it's O-level cost, match everyone in that office
- If it's FW-level cost, match everyone

And the critical calculation:
```sql
CostWorked = Q.CostWorked * H.Weight
```

That weight we calculated earlier - that's where it gets applied!

So if we have $100,000 in NY Litigation salary costs, and Sarah's weight is 23.5%, she gets $23,500. The system creates one row per person, and the sum of all those rows equals the original $100,000.

That's allocation. Costs flow from ProfitCostRaw (at various levels) to ProfitCostWa (all at TK level).

---

## **SLIDE 8: Allocation Factor Slots (0-9)**

**[SPEAKER NOTES]**

Let me show you something powerful. Look at the ProfitHr table structure.

We have AllocationFactor0 through AllocationFactor9. Why 10 slots?

Because you might want different allocation schemes for different types of costs:

**AllocationFactor0 (ALLTK)**: 
- Used for most costs
- Based on FTE × Title weight
- Requires 100+ billable hours

**AllocationFactor1 (BILLHRS)**: 
- Could be based purely on billable hours
- No title weighting
- Useful for allocating office supplies, IT costs

**AllocationFactor2 (REVENUE)**:
- Based on revenue generation
- Partners who bring in $5M get more weight than those who bring in $1M
- Useful for allocating bonus pools

**AllocationFactor3 (SENIORITY)**:
- Years of experience based
- Useful for allocating mentoring costs

You can define these however makes sense for your firm.

In the code, we currently only define AllocationFactor0. But the infrastructure supports 10 different schemes. When you create a new allocation factor, you pick an empty slot and define your calculation logic.

That's what we'll do in the live demo - create AllocationFactor1.

---

## **SLIDE 9: The ProfitAccountVariant Control Table**

**[SPEAKER NOTES]**

Now, how does the system know which allocation factor to use for which account?

That's controlled by this table: #ProfitAccountVariant

*[CLICK to show code snippet]*

Look at line 1080. This temporary table maps profit accounts to allocation methods.

For example:
```
'COMP-SALARY.0',  'ALLTK',  'OVERHEAD.0',  '',  '',  'TK',  'OD',  'O',  'D',  'FW'
```

This says:
- Account: COMP-SALARY.0 (salary costs)
- AllocationFactor: ALLTK
- ResidualAccount: OVERHEAD.0 (if there's leftover variance)
- Then sources: where this cost can come from
  - Source_GL_TK: 'TK' (can come from GL at TK level)
  - Source_GL_PP: 'OD' (can come from GL at OD level if practicing/practicing)
  - Source_GL_PS: 'O' (can come from GL at O level if practicing/support)
  - And so on...

This is your control panel. When you want to:
- Add a new profit account
- Change which allocation factor it uses
- Change what granularities are allowed

You modify this table.

It's defined in code right now (around line 1087), but at some firms, this gets externalized to a configuration table so finance can manage it without changing code.

---

## **SLIDE 10: Granularity Hierarchy**

**[SPEAKER NOTES]**

Let's talk about granularity more deeply, because this is where people get confused.

Think of granularity as a hierarchy:

**Most Specific:**
- **TK** (Timekeeper) - "This $50K is specifically for John Smith"

**Mid-Level:**
- **OD** (Office+Department) - "This $200K is for NY Litigation department"
- **O** (Office) - "This $500K is for the entire NY office"
- **D** (Department) - "This $300K is for all Litigation departments firm-wide"
- **T** (Team) - "This $100K is for the M&A team"

**Least Specific:**
- **FW** (Firm-Wide) - "This $1M is for the entire firm"

There are also compound granularities:
- **C** (Company) - for multi-entity firms
- **OG** (Office Group) - regional groupings
- **DG** (Department Group) - practice area groupings
- **CD** (Company+Department) - entity-specific department costs

The rule is: **Allocate at the most specific granularity possible.**

Why? Accuracy. 

If you know NY Litigation spent $200K on salaries, allocate it to NY Litigation TKs only. Don't allocate it firm-wide - that would give credits to people in the SF Corporate department who had nothing to do with that cost.

But if you have a firm-wide insurance policy for $1M and you literally cannot break it down by office, then FW is appropriate.

The system handles all of this automatically based on:
1. Where the cost exists in the source data
2. Where people exist in the HR data
3. The matching logic we saw earlier

---

## **SLIDE 11: Common Pitfalls**

**[SPEAKER NOTES]**

Alright, let's talk about where things go wrong. I've debugged dozens of allocation issues, and they almost always come down to these four problems:

**Pitfall #1: Zero Division Errors**

This happens when AllocationFactorTotal = 0. 

Example: You have GL costs in the "Support Services" department with granularity OD. But everyone in Support Services has NULL for AllocationFactor0 because they don't meet the 100 billable hours requirement.

Result: No one to allocate to. The costs sit in ProfitCostRaw but never make it to ProfitCostWa.

Fix: Either exclude support departments from the allocation, or create a different allocation factor (like AllocationFactor1 based on headcount) for support costs.

**Pitfall #2: Support Unit Contamination**

You define an allocation factor that includes support staff, then wonder why your TK profitability is being dragged down.

Example: You forget the `AND O.Type <> 'Support'` filter when calculating AllocationFactor0.

Result: The office administrator with zero billable hours gets allocated a share of partner compensation costs.

Fix: Always filter to practicing units when defining revenue-based allocation factors.

**Pitfall #3: Granularity Mismatch**

You have TK-level costs (specific person salaries) but try to allocate them at FW granularity.

Result: John's $150K salary gets spread across the entire firm instead of allocated to him specifically.

Fix: Check the SourceId in ProfitCostRaw. If it says 'IMP-TK' or 'GL-TK', make sure granularity is 'TK' and the WorkingAttorneyId is populated.

**Pitfall #4: Missing Filters in Joins**

The most subtle one. Look at this join again:
```sql
H.Granularity = 'OD' AND H.OfficeId = Q.OfficeId 
                     AND H.DepartmentId = Q.DepartmentId
```

If you forget one of those conditions, you'll allocate costs to the wrong people.

Example: You match on OfficeId but forget DepartmentId. Now NY Litigation costs get allocated to NY Corporate TKs too.

Fix: Double-check the matching logic for every granularity level.

---

## **SLIDE 12: Debugging Tips**

**[SPEAKER NOTES]**

When something goes wrong - and it will - here's how you debug it.

**Query 1: Check Allocation Factor Coverage**

*[CLICK to show query]*

```sql
SELECT PeriodId, 
       COUNT(*) AS TotalPeople,
       SUM(CASE WHEN AllocationFactor0 IS NOT NULL THEN 1 ELSE 0 END) AS HasALLTK
FROM bi.ProfitHr
GROUP BY PeriodId
```

This tells you: Out of 500 people in the firm, how many have ALLTK defined? If it's only 150, you know 350 people won't receive allocations. Is that expected?

**Query 2: Verify Weights Sum to 1.0**

*[CLICK to show query]*

```sql
SELECT PeriodId, Granularity, SUM(Weight) AS TotalWeight
FROM bi.ProfitHrTemp
WHERE AllocationFactor = 'ALLTK'
GROUP BY PeriodId, Granularity
```

For each period and granularity, weights should sum to 1.0 (or 100%). If you see 0.85, you know 15% of costs will be orphaned.

**Query 3: Trace a Specific Cost Allocation**

*[CLICK to show query]*

```sql
-- Before allocation
SELECT * FROM bi.ProfitCostRaw 
WHERE ProfitAccountId = 'COMP-SALARY.0' 
  AND PeriodId = 202501

-- After allocation
SELECT * FROM bi.ProfitCostWa
WHERE ProfitAccountId = 'COMP-SALARY.0' 
  AND PeriodId = 202501
```

Compare the totals. They should match. If you have $1M in ProfitCostRaw and only $850K in ProfitCostWa, you have a problem.

Then drill in: Who got allocated what? Does it make sense?

**Pro Tip:** Add up CostWorked by SourceId:
```sql
SELECT SourceId, SUM(CostWorked)
FROM bi.ProfitCostWa
WHERE ProfitAccountId = 'COMP-SALARY.0' AND PeriodId = 202501
GROUP BY SourceId
```

If you see unexpected SourceIds (like 'GL-WASHUP' when you expected 'IMP-TK'), that's a clue something got treated as a residual.

---

## **SLIDE 13: Best Practices**

**[SPEAKER NOTES]**

Let me leave you with five best practices before we move to the live demo.

**1. Define Clear Business Rules**

Document exactly what each allocation factor means:
- What's included, what's excluded
- What filters apply
- Why you made those choices

Write it down. Future you will thank present you.

**2. Use Appropriate Granularity**

Don't allocate everything firm-wide just because it's easier. Push down to the most specific level possible.

But also don't force artificial precision. If you truly can't break down a cost beyond the office level, don't fake OD granularity.

**3. Validate Weight Totals**

After defining a new allocation factor, run the weight validation query. Make sure weights sum to 1.0 at every granularity where you'll use that factor.

**4. Handle NULLs Gracefully**

Decide upfront: Do people with NULL allocation factors:
- Get excluded from allocations? (Usually correct)
- Get a default value of 1.0? (Sometimes needed for support staff)
- Cause an error? (Useful during testing)

Code it explicitly. Don't rely on implicit SQL NULL behavior.

**5. Test with a Small Dataset First**

When creating a new allocation factor:
1. Test with one period
2. Test with one office
3. Verify the math manually in Excel
4. Then run firm-wide

It's much easier to debug 10 rows than 10,000 rows.

---

## **SLIDE 14: Architecture Diagram**

**[SPEAKER NOTES]**

*[CLICK to show diagram]*

Let me tie this all together with a visual.

**[Point to diagram]**

Source data flows in from three places:
1. **ProfitGl** - General Ledger costs
2. **ProfitCostImport** - Externally imported costs (like detailed comp data)
3. **ProfitRevenue** - Revenue that acts as contra-cost (like NTK revenue)

All of these land in **ProfitCostRaw** at various granularities (TK, OD, O, FW, etc.).

Meanwhile, **ProfitHr** defines allocation factors for each person.

The **ProfitProcess** stored procedure:
1. Reads ProfitHr
2. Calculates weights in ProfitHrTemp
3. Reads ProfitCostRaw
4. Joins them together
5. Multiplies Cost × Weight
6. Writes to ProfitCostWa

From there, costs flow to **ProfitCost** (allocated to matters) and eventually to the final **Profit** output that users see in reports.

The allocation factors are the engine that powers steps 2-5.

---

## **SLIDE 15: Summary**

**[SPEAKER NOTES]**

Alright, let's recap what we've covered in this technical deep dive:

✅ **Allocation factors use weight-based distribution**: Value divided by Total equals Weight

✅ **Four key tables**: ProfitHr stores values, ProfitHrTemp calculates weights, ProfitCostRaw holds unallocated costs, ProfitCostWa holds allocated costs

✅ **The flow**: Define values → Calculate totals → Load raw costs → Allocate to TKs

✅ **Granularity matters**: Allocate at the most specific level possible (TK, OD, O, FW, etc.)

✅ **10 allocation factor slots**: AllocationFactor0 through 9 let you define multiple schemes

✅ **Common pitfalls**: Zero divisions, support unit contamination, granularity mismatches, missing join filters

✅ **Debug with queries**: Check coverage, verify weights sum to 1.0, trace specific costs

Any questions before we move to the live demo?

*[Pause for questions]*

Great! Now let's see this in action. We're going to create a brand new allocation factor from scratch, test it, and validate the results.

---

## **TRANSITION TO STEP 4**

**[SPEAKER NOTES]**

In the next section, I'll create "AllocationFactor1" - a billable hours-based allocation. We'll:
- Write the SQL to calculate it
- Add it to ProfitHrTemp
- Configure a profit account to use it
- Run the process
- Validate the numbers

Let's switch to the live environment...

---

## **[END OF SCRIPT]**

---

## **SPEAKER TIPS:**

- **Pace:** Slow down on the code sections. Let people absorb it.
- **Interaction:** Ask "Does this make sense?" after complex concepts
- **Analogies:** Use real-world examples (pizza slices, budget splitting)
- **Energy:** Maintain enthusiasm - technical doesn't mean boring!
- **Questions:** Pause after each major section for questions
- **Backup Slides:** Have extra detail slides for deep questions

---

## **ESTIMATED TIMING:**

- Slides 1-2: 2 minutes
- Slides 3-7: 5 minutes (the core flow)
- Slides 8-10: 2 minutes
- Slides 11-12: 2 minutes
- Slides 13-15: 1 minute

**Total: 12 minutes** (within the 10-12 minute target)