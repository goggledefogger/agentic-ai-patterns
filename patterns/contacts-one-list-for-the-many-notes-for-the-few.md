---
type: pattern
date: "2026-10-23"
source: A personal agent importing a 694-row phone-book export into the personal vault, so the router agent could tell who someone is before routing
tags:
  - vault
  - privacy
  - people
  - routing
---

# Contacts: One List for the Many, a Note for the Few

A phone-book export holds hundreds of people. A vault's People folder usually holds a dozen, the ones with history worth writing down. Importing the first into the second as one note per contact buries the dozen under hundreds of empty stubs, and every stub becomes a dot on the graph that means nothing.

## The Pattern

**1. The personal vault owns contacts. Other agents keep a lookup, never a copy.** A router that needs to know who someone is asks the vault and gets back a pointer plus the few fields routing needs. If the router keeps its own copy, there are two lists and one of them is wrong.

**2. The long tail is one data file. A person gets a note when they start to matter.** Keep the export as a single CSV beside the People notes (`People/_originals/contacts.csv`). Write a note only when there is history, a decision, or a relationship to record. Lookups search the notes first, then the list, so a note always outranks its row.

**3. Split by sensitivity at the field level, not the file level.** A contact list mixes two kinds of data:

| Fields | Tier | Why |
|---|---|---|
| name, nickname, org, email, phone | routine | what any lookup needs, and what everyone already hands out |
| addresses, birthdays, notes, relations, photos | confidential | where someone lives, and what you wrote about them |

Vault tiers attach to files, so the split has to be two files. The routine one is a plain CSV under the folder's default tier. The confidential one carries its tier in frontmatter (a markdown file wrapping the CSV in a fenced block) so the same tooling that reads every other note's tier reads this one too. A CSV has nowhere to declare a tier, so it falls back to the folder default.

**4. The lookup returns less than the list holds.** The router's person lookup returns name, org and email. It returns no phone and no address. Routing needs to know who someone is and which part of life they belong to, not how to reach their front door.

**5. Replace the list whole on each import, and let version control be the change log.** The importer writes the full list fresh and prints who was added, removed or changed since the last run. A live sync later (the provider's change token, polled by a plain script with no model) writes the same file, so manual and automatic imports share one shape.

## Why It Works

- One file per tier keeps the access rules where the rest of the vault already enforces them, instead of a new per-field gate nobody else knows about
- The graph stays honest. A dot on the map means a person with history, not a row from 2015
- The router stays light. It holds a query, and the vault stays the only place a contact is edited

## The Trap the Split Introduces

The change report has to compare only the routine fields. The first version compared whole rows. That worked against the old full copy and would have flagged all 694 contacts as changed on the next import, because the stored file no longer had the confidential columns. A test built only from same-shaped files passes anyway. The fixture needs an import where the stored shape differs from the export, and a second identical import that must report nothing.

## A Residual the Split Does Not Close

A name field can carry sensitive information by itself. People label contacts "Sam (Counselor)" or "Nurse Line, Insurer", and that sits in the routine file. Field splitting moves the obvious fields. It does not read what a person typed into a name. Say so where the tier is documented, and treat the routine file as not demo-safe even when it is fine for an agent to read.

## When to Use

- Bringing any address book (phone, email provider, social network export) into a vault that already has per-person notes
- Any agent that has to know who a person is before deciding where a message or task goes

Skip it when the list is small enough that every entry deserves a note anyway.

## Adjacent Patterns

- [[sensitivity-tiered-access-control]] covers the tiers this pattern splits across
- [[external-store-as-source-of-truth]] is the same idea at the vault level: one owner, pointers everywhere else
- [[wikilink-graph-hygiene]] explains why empty stubs cost more than they look
- [[a-heuristic-where-an-exact-key-exists]] applies here too: resolve a person to their email through the lookup, never by searching for their name
