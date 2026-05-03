---
description: Execute a deep refinement of an existing module documentation index (Index, FAQ, QA Scenarios) to enterprise standards.
---

# Refine Documentation

Run this workflow AFTER an initial `brand_module` (or equivalent module
documentation scaffold) sequence — and preferably in a fresh session to
maximise the available context window — to elevate the documentation
quality.

## Refinement steps

Apply the following refinements to the documentation index. If the index
is very large, split the changes into a plan and implement it in parts so
the agent retains adequate context per pass.

The refinements are surgical: only the named sections of the index move;
the rest of the index is preserved verbatim.

1. **Technical Support section.** Verify that the index ends with a
   "Technical Support" section. If missing, append it using the team's
   standard contact block.

2. **Setup & User Manual section.** Make sure this section is comprehensive,
   detailed, and descriptive. The configuration part must contain a
   step-by-step guide so detailed it is impossible to get wrong, and must
   spell out the side-effects of each configuration choice. The usage /
   workflow part must cover every relevant aspect of the application. The
   target is that a customer reading only this section never needs an
   external manual or support contact for functional questions. If during
   the working session the user asked questions that revealed gaps in
   understanding, ensure those questions are now answered explicitly here.

3. **QA / User Testing Scenarios section.** Without removing the existing
   content, add many additional scenarios that exercise real-world cases
   covering the application end-to-end at enterprise grade.

4. **FAQ & Troubleshooting section.** Without removing the existing content,
   add more blocks covering the most likely frequently-asked questions and
   the common problems that may arise, with concrete resolution steps.
