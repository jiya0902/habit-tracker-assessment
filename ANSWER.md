# ANSWERS.md

## 1. How to run

### Requirements

* Node.js installed
* npm installed

### Steps to run locally

```bash id="sl1q5u"
git clone https://github.com/jiya0902/habit-tracker-assessment.git
```

```bash id="m2w09l"
cd habit-tracker-assessment

After running the command, open:

```text id="n7jyvf"
http://localhost:5173
```



# 2. Stack & design choices

I chose React with Vite and Tailwind CSS because it provides fast development, reusable components, and responsive UI styling with minimal setup. React made it easier to manage the habit state, week navigation, and localStorage updates in a clean way.

One important design decision was using a weekly grid instead of a vertical checklist. Habit tracking is easier when users can scan progress horizontally across days, so the grid layout improves quick visual understanding of streaks and missed days.

Another design decision was highlighting today’s column with a stronger background and border color. This creates a clear visual anchor so users instantly know which day they are interacting with without reading the dates repeatedly.

I also chose Monday as the start of the week because it creates a more structured work-week flow and keeps weekends grouped together visually.

For streak logic, I counted streaks up to yesterday if today is unchecked. This avoids unfairly breaking a streak early in the day before the user has completed the habit.

---

# 3. Responsive & accessibility

On a 360px mobile screen, the app becomes horizontally scrollable so the weekly grid remains readable without shrinking the cells too much. Spacing and typography are reduced slightly for smaller devices, and the habit controls stack vertically for easier tapping.

On a 1440px desktop screen, the layout expands with more spacing, larger habit rows, and a centered container to improve readability and visual balance.

For accessibility, I added keyboard-accessible buttons and visible focus states so users can navigate the interface without a mouse. I also ensured sufficient color contrast between text and background elements.

One accessibility improvement I knowingly skipped was full screen-reader optimization with ARIA live announcements for habit toggles. I prioritized completing the core interaction flow first because of time constraints.

---

# 4. AI usage

I used ChatGPT and Claude AI during development.

### ChatGPT

Used for:

* Planning the project structure
* Generating frontend workflow ideas
* Writing README.md and ANSWERS.md guidance
* Debugging React and Tailwind issues

### Claude AI

Used for:

* Generating React components
* Creating Tailwind UI layouts
* Building the weekly habit grid
* Implementing localStorage persistence
* Creating streak calculation logic

One specific change I made to the AI-generated output was modifying the grid responsiveness. The AI initially generated fixed-width columns, which caused layout overflow on smaller screens. I replaced that approach with a horizontally scrollable responsive grid and flexible sizing so the layout remained usable on narrow mobile devices.

I also adjusted the streak logic manually because the original implementation immediately reset streaks if the current day was unchecked. I changed it so the streak continues until the day ends, which feels more natural for users.

---

# 5. Honest gap

One part of the project that still needs improvement is the animation and interaction polish. While the interface is functional and responsive, the checkmark interactions and transitions could feel smoother and more engaging.

With another day, I would improve:

* Micro-interactions when toggling habits
* Better loading and empty-state animations
* Drag-and-drop habit reordering
* More advanced accessibility support
* Better touch gestures for mobile users

I would also add automated testing for streak calculations and week navigation to improve reliability.

