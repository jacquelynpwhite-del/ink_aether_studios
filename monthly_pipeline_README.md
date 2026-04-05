# Reusable Monthly Pipeline System (Google Sheets)

This version is designed so you only enter:
- **Book Title**
- **Pen Name**
- **Release Month**

Everything else is auto-derived (drafting, revision, production, launch checks, and overload warnings).

## 1) Never-changing monthly structure
Keep `Monthly_Pipeline` columns fixed forever:
1. Month
2. Pen Name
3. Book Title
4. Phase
5. Week Breakdown
6. Status

This structure does not change year to year. You only swap release assignments.

## 2) Input table (where you edit each year)
Use `Release_Assignments` with this layout (or import `release_input_template.csv`):
- `A: Book Title` (input)
- `B: Pen Name` (input)
- `C: Release Month` (input, first day of month)
- `D: Drafting Months` (auto)
- `E: Revision Weeks (8-10)` (auto)
- `F: Production Month` (auto)
- `G: Cadence Check` (auto)
- `H: Overload Check` (auto)
- `I1: Max Concurrent Books` (capacity setting, e.g. `3`)

## 3) Formulas (clear logic)

### D2 — auto-assign Drafting months (4-6 weeks)
```gs
=IF(C2="","",TEXT(EDATE(C2,-5),"mmm yyyy")&" + "&TEXT(EDATE(C2,-4),"mmm yyyy"))
```

### E2 — auto-assign Revision weeks (8-10 weeks: drafts 2-7)
```gs
=IF(C2="","","D2-D5 in "&TEXT(EDATE(C2,-3),"mmm yyyy")&"; D6-D7 + buffer in "&TEXT(EDATE(C2,-2),"mmm yyyy"))
```

### F2 — auto-assign Production month (4 weeks)
```gs
=IF(C2="","",TEXT(EDATE(C2,-1),"mmm yyyy"))
```

### G2 — enforce fixed pen-name release cadence
```gs
=IF(C2="","",IF(
 OR(
  AND(B2="J.P. White",OR(MONTH(C2)=1,MONTH(C2)=7)),
  AND(B2="Jade Black",OR(MONTH(C2)=3,MONTH(C2)=4,MONTH(C2)=11)),
  AND(B2="Indigo Winter",OR(MONTH(C2)=5,MONTH(C2)=6,MONTH(C2)=9,MONTH(C2)=10))
 ),
 "OK",
 "INVALID"
))
```

### H2 — overload prevention (warn if capacity exceeded)
Checks each month in the 6-month window (`Release-5` through `Release`) and flags overload when concurrent books exceed `I1`.

```gs
=IF(C2="","",IF(
 MAX(
  MAP(
   SEQUENCE(6,1,0,1),
   LAMBDA(n,
    COUNTIFS($C$2:$C$200,">="&EDATE(C2,-5+n),$C$2:$C$200,"<="&EDATE(C2,n))
   )
  )
 )>$I$1,
 "OVERLOAD",
 "OK"
))
```

Copy `D2:H2` down for all release rows.

## 4) Monthly phase auto-alignment logic
In `Monthly_Pipeline`, each row gets phase by months-to-release from the same book title:
- `m = 0` → Launch (2 weeks)
- `m = 1` → Production (4 weeks)
- `m = 2-3` → Revision (8-10 weeks)
- `m = 4-5` → Drafting (4-6 weeks)

Phase formula (example for `Monthly_Pipeline!D2` where `A2=Month`, `C2=Book Title`):

```gs
=IF(OR(A2="",C2=""),"",LET(
 rel, XLOOKUP(C2, Release_Assignments!$A:$A, Release_Assignments!$C:$C, ""),
 m, (YEAR(rel)-YEAR(A2))*12 + MONTH(rel)-MONTH(A2),
 IF(m=0,"Launch",IF(m=1,"Production",IF(OR(m=2,m=3),"Revision",IF(OR(m=4,m=5),"Drafting",""))))
))
```

Pen name formula (`Monthly_Pipeline!B2`):

```gs
=IF(C2="","",XLOOKUP(C2,Release_Assignments!$A:$A,Release_Assignments!$B:$B,""))
```

Week breakdown formula (`Monthly_Pipeline!E2`):

```gs
=IF(D2="","",SWITCH(D2,
 "Drafting","W1 Outline; W2 Draft 25%; W3 Draft 50%; W4 Draft 75% / W5 Draft 100%; W6 Cleanup + notes",
 "Revision","W1 Dev edits (D2); W2 Line edits (D3); W3 Copy edits (D4); W4 Proof (D5) / W5 Beta pass (D6); W6 Final polish (D7); W7 Prep handoff; W8 Buffer",
 "Production","W1 Typeset; W2 Cover + metadata; W3 Upload QA; W4 ARC send",
 "Launch","W1 ARC + release prep; W2 Release + promo",
""))
```

## 5) Included files
- `release_input_template.csv`: one-row formula template for input logic.
- `release_assignments_example.csv`: filled example entries (2026 and 2027).
- `monthly_pipeline_template.csv`: expanded monthly output showing overlapping phases.

## 6) Reuse each year without rebuilding
1. Keep the same tabs/columns/formulas.
2. Replace only `A:C` values in `Release_Assignments` for the new year.
3. Confirm `G` = `OK` (cadence valid) and `H` = `OK` (no overload).
4. `Monthly_Pipeline` phase mapping remains automatic from release month.
