# Osysharp.Automations

Rules that do an app's routine work when something happens — "when a customer writes on a solved ticket, set it Open
and tell whoever holds it"; "when a job has waited two days in Ready, label it uncollected". A rule is a trigger,
conditions and steps, run in order, and every run is written where the record's history is read.

The kit never knows what the rules are about, and the app does not tell it. What happens to a record is its events,
what can be done to it is its `[Action]` functions, and what a rule may ask of it is its own fields — so a record that
has events and actions has rules, with nothing more to write.

```osy
use Osysharp.Automations@0;

policy RunsAutomations => IsAgent;        // who works with rules, and keeps their own
policy ManagesAutomations => IsAdmin;     // who keeps the team's, and copies and locks the app's

[Event("A ticket arrives")] public sealed class TicketArrived { public Ticket Ticket; }   // a trigger
Raise(new TicketArrived { Ticket = t });                                                 // where it happens

[Action("Set priority")]                                                                 // a step
public string SetTicketPriority(Ticket ticket, WorkPriority priority) { … }

// Only to add what the model does not say — the rules the app ships with, a computed condition, the hourly clock:
app.Automations = new AutomationsSetup { Subjects = [new TicketRules()] };
public class TicketRules : AutomationSubject<Ticket> {
  public override List<AutomationDefault> Defaults() { return [
    new AutomationDefault { Key = "urgent", Name = "Urgent words are tagged", When = On<TicketArrived>(),
                            If = [Test(t => t.Title, Contains, "urgent"),               // field and value: checked
                                  Test(t => t.Priority, IsNot, WorkPriority.Urgent)],
                            Then = [Run(t => TagTicket(t, "urgent"))] },                // the action, called: checked
  ]; }
}
```

## What it does

| | |
|---|---|
| **Triggers** | every event about the record — the app's own and any kit's, each by its label — or **every hour**, over what the app's subject is `Watching()` |
| **Costs nothing it does not use** | the kit hears only the events some rule that is on runs on: an event no rule is on is not even written, so an app with no rules pays for its events what it paid without the kit. A rule saved now runs on the very next event, and the rules of the events one commit raised are read once between them |
| **Conditions** | the record's own fields, each compared by its kind: a choice (an enum's members, a small table's rows), words, a number, yes/no, or **hours since** a time ("in its status for at least 72 hours"); only the fields the rule's author may read, and never one marked `[NotInRules]`. A condition the app computes (its tags, hours since the customer wrote) is a class of its own beside them. The app's own rules are typed throughout — a field by a selector, a value of the field's own type (`Test(t => t.Priority, IsNot, WorkPriority.Urgent)`) — so a misspelt field or a value of the wrong type does not compile. An hourly rule runs once per wait, not every hour of it |
| **Steps** | the app's `[Action]` functions on the record, with the choices each offers (an enum's members; a record's from the function it names); the first refused stops the rest. The app's own rules write a step as the action called — `Run(t => TagTicket(t, "urgent"))` — so a misspelt parameter or a value of the wrong type does not compile |
| **Three layers** | the app's rules from code (they keep the app's updates); the team's, and a team **copy** of one of the app's that replaces it for everyone; each person's own, running on what they hold, and their own copies. A copy can be **restored**; one of the app's can be **locked** so nobody replaces it |
| **Who it runs as** | a person's rule, and the team's, as the person who saved it, under their own security when it runs — saving records their consent to each step. The app's rules run as the app |
| **Never a loop** | a rule runs once in a chain of its own making; a rule set off again by another is stopped, a chain past `MaxChain` too — each stop written down |
| **On the record** | every run raises `AutomationRan`, which says itself — "Rule “Refunds go high” (as Ada Lindgren): Set priority: High." — on the record's timeline (`Osysharp.RecordTimeline`) or wherever the app's own `[On]` handler writes it, and is listed under "What ran lately" for those who may read the record |
| **The page** | `/settings/automations` in the app's settings area: the rules in the order they run, copies under the rule they replace, the editor (when, only if, then), what ran lately |

## Its tests

The kit's test app is a print shop — jobs placed, printed and made ready for collection — so the kit is shown working
outside a helpdesk: rules on their trigger and in order, loops stopped, a rule running as its saver and refused when
they may not, conditions only on what the author may read, the three layers, locks, the hourly clock, and the page,
wide and on a phone, light and dark — run with `osy test`, and photographed in a real browser with
`osy test --pixels`.
