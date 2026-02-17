I've linked the poster that my team presented at the SURP Symposium below, and it's a good overview, but the writeup just beneath that certainly goes into more depth. Both provide a good overview of the project, so its a bit of a pick your poison decision. 

![[CENGPOSTER.pdf]]
<h3>Intro / Goals</h3>

  

<p>The problem statement for this project was clear, we wanted to make the tedious and often conflict inducing process of assigning students to project teams easy. We had to be able to respect who wants which project, who wants to work together (or not), and keep group sizes even all at once. We also wanted the tool to be able to take a survey of preferences and “who to team with,” as this was the existing input for this process. The tool would then automatically propose good team assignments and let instructors or admins explore trade-offs instead of guessing in a spreadsheet.</p>

<p>I worked from an existing codebase that had grown out of architecture-design work (Vassar/Jess and MATLAB) from my advisor/mentor's own collegiate work and research. The goal on my side was to adapt and extend it into a usable auto-grouping tool that had better default parameters, clearer handling of group-size constraints, and a more practical workflow. This included making a GUI for loading rules, running the optimizer, and inspecting results.</p>


<h3>What the Tool Does</h3>
<p>AutoGroup is a MATLAB application that assigns students to project teams using a multi-objective genetic algorithm. It optimizes two objectives at once:</p>
<ul>

<li><strong>Project preference score</strong> — How well each student’s assigned project matches their stated preferences (from the survey).</li>

<li><strong>Member synergy score</strong> — Bonuses for placing “preferred teammates” together and penalties for placing “do not team with” pairs in the same group (also derived from the survey.</li>

</ul>
<p>Group sizes are enforced via a penalty so that teams stay close to an ideal size set by the user (e.g., 4). The solver produces a set of Pareto-optimal assignments which allows the user to then pick a solution that balances preference satisfaction and team member synergy (from a scatter plot in the GUI). Input is an Excel rulesheet (students × projects, preferences, and an optional “member_pref” sheet for preferred/do-not-team). Output is the same data plus Pareto solutions and, in the GUI, interactive selection and export. Under the hood, the project preference score is computed by a Jess rule engine (Vassar-style). The spreadsheet is turned into rules, and each candidate assignment is asserted as a fact and evaluated. That keeps the logic flexible and consistent with the original research stack.</p>

<h3>Tech Stack</h3>

  

<table>

<thead>

<tr>

<th>Category</th>

<th>Technology</th>

</tr>

</thead>

<tbody>

<tr><td>Language / environment</td><td>MATLAB</td></tr>

<tr><td>Optimization</td><td>Multi-objective GA (<code>gamultiobj</code>), integer chromosome (student → project index)</td></tr>

<tr><td>Preference scoring</td><td>Jess rule engine (Vassar <code>templates.clp</code>, generated <code>prefrules.clp</code> / <code>aggrules.clp</code>)</td></tr>

<tr><td>Input / output</td><td>Excel rulesheets (preferences + optional synergy sheet)</td></tr>

<tr><td>UI</td><td>MATLAB App (scatter plot, team builder table, load/run/export)</td></tr>

</tbody>

</table>

  

<h3>Changes I Made (Contributions)</h3>

  

<p>Below are the main areas I improved this tool upon.</p>

<h4>1. Research-based default parameters</h4>
<p>When a rulesheet is loaded in the GUI, population size, number of generations, and penalty weight are no longer fixed magic numbers. They are set from formulas tied to the size of the problem (number of students and projects), as suggested in the research materials:</p>
<ul>

<li>Population size: <code>125 × log10(students) × log10(projects)</code></li>

<li>Generations: <code>20 × log10(students) × projects</code></li>

<li>Penalty weight: <code>14 × (students − projects)</code></li>

</ul>


<p>So the GA scales with class and project count instead of using one-size-fits-all defaults. This lives in <code>loadRules.m</code> (under the “Continue” branch when loading a new rule file).</p>
<h4>2. Stronger, cubic penalty for group-size violation</h4>
<p>Group-size feasibility is enforced by penalizing solutions whose team sizes deviate from the ideal. I replaced the earlier penalty with a <strong>cubic</strong> deviation term so that large deviations are punished much more than small ones:</p>
<ul>

<li>For each team, compute deviation from ideal size (only for non-empty teams).</li>

<li>Penalty term: <code>(sum(deviation³) + 3) × penalty_weight / number_of_teams</code>.</li>

</ul>
<p>This is applied both in the main GA loop (<code>GA_autoGroup.m</code>) and in the fitness evaluation (<code>fordff.m</code>). I also added bounds checks so that group indices stay valid (1 to number of projects) and don’t cause silent indexing errors.</p>

<h4>3. Feasibility when class size doesn’t divide evenly</h4>

<p>When the number of students isn’t a multiple of the ideal group size, you can’t have every team exactly at the ideal. The code now treats a solution as <strong>feasible</strong> when:</p>
<ul>

<li>If there is a remainder (students mod ideal ≠ 0): every non-empty team is either at the ideal size or one member short (ideal − 1), as needed to account for the remainder.</li>

<li>If there is no remainder: every non-empty team is exactly the ideal size.</li>

</ul>

<p>So “one member away” is explicitly allowed when the math requires it. This logic appears in <code>GA_autoGroup.m</code> and <code>evaluate_score.m</code> and is used to mark solutions as feasible and to highlight them in the UI (e.g., stars on the scatter plot).</p>

<h4>4. GUI and workflow</h4>
<p>I extended the GUI so that the user can:</p>

<ul>

<li>Load a rulesheet from a <code>rules/*.xlsx</code> file and have the research-based parameters filled in automatically.</li>

<li>See a <strong>Team Builder</strong> table: rows = students, columns = projects, with optional project ID shortening (e.g., strip suffix after <code>_</code> for readability).</li>

<li>View a <strong>scatter plot</strong> of the two objectives (preference vs. synergy), with feasible solutions highlighted and click-to-select to see the corresponding assignment in the table.</li>

<li>Use a toggle (e.g., “All” vs. filtered) to show only solutions with low penalty when desired.</li>

<li>Reset state (clear population, scores, plot, table) before loading new rules or re-running.</li>

</ul>
<p>Styling in the Team Builder (e.g., green for “correct group size,” blue for preference cells, and labels like “(n=4)” and “(pref=3)”) were added so users can quickly see how well a selected solution satisfies preferences and group sizes. The scatter plot’s click callback finds the nearest point and updates the builder table to that architecture.</p>

<h4>5. Synergy preprocessing and name matching</h4>

<p>The synergy matrix (who wants to work with who) is built from a “member_pref” sheet (e.g., PreferredTeam, DoNotTeam columns). Names in that sheet often don’t match the main roster exactly, so I used <strong><code>strnearest</code></strong> (Levenshtein-based, with optional case-insensitive matching) to map free-text names to the canonical student list before building the synergy matrix in <code>createSynergy.m</code>. That makes the tool more robust to real life data with typos or formatting issues.</p>

  

<h3>Workflow / Outcome</h3>

  

<p>In practice, a professor or admin can:</p>
<ol>

<li>Export survey data into the Excel rulesheet format (and optionally the member_pref sheet).</li>

<li>Open the app, load the rulesheet, and (optionally) tweak ideal group size or penalty.</li>

<li>Run the GA; the app shows the Pareto front and feasible solutions.</li>

<li>Click points on the scatter plot to inspect different trade-offs and pick an assignment.</li>

<li>Export the chosen assignment (e.g., to Excel) for publishing or further editing.</li>

</ol>

  

<p>The combination of multi-objective search, explicit feasibility handling, and research-based defaults makes it possible to get good, balanced team assignments without hand making groups and tuning for every new class. The Jess integration keeps the preference logic transparent and easy to extend (e.g., different scoring schemes) without rewriting the entire project as a whole.</p>

  


<h2>Conclusion</h2>
This project was a massive undertaking for me at a young point in my career, but I consider it one of the greatest growth opportunities I've had thus far. Getting to work hand in hand with a mentor/advisor and build upon the work of another developer (in this case that very same mentor) was an experience that I feel has been valuable in my industry work thus far and likely will continue to be. The core work that many of us developers due is tantamount to standing on the shoulders of all those that have come before us, and placing another brick on the foundation that our predecessors have laid before us. This was the greatest element I took away from this project. 