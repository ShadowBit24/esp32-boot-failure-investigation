# Lessons Learned — ESP32 Multi-Board Boot Failure Investigation

Transferable lessons, stated generally because they apply to any board bring-up or validation task, not only to this batch of ESP32 boards.

---

## 1. "Flashes and verifies" validates the downloader path, not the boot path

esptool write/verify exercises the ROM downloader: host-driven, host-paced, host-checked. Autonomous boot exercises the ROM's own read of the same flash under boot parameters. They share silicon and pins and almost nothing else that matters. In this investigation one path was flawless and the other was broken. Acceptance criteria for any board must include *executing* a known image, not only flashing one.

## 2. Non-deterministic wrong data is a different fault class from constant wrong data

A constant wrong header means corrupted content. A wrong header that changes between runs, and changes again when interface settings change, means the content is fine and the *transfer at that moment* is unreliable. Reading the symptom at that resolution redirected this investigation from "what is in the flash" to "what happens when the ROM reads the flash at boot", which is where the fault actually lives. Record exact error values, every time; the pattern across values is data.

## 3. Change one variable, record, repeat

SPI mode and clock were changed individually, with the header value logged per configuration. Because of that discipline, three test runs produced a conclusion (interface-parameter-dependent corruption) instead of three more anecdotes. Changing two settings at once would have destroyed the signal.

## 4. Check security state early on any board that behaves inconsistently

The eFuse audit took minutes and reframed the investigation: an active flash-encryption counter, encryption configuration, and read-disable state explained the erase refusal and put the security/configuration domain inside the fault boundary. On any unfamiliar or suspect board, capture the security/fuse summary at first power-on, before anything writes to the device. It is part of the board's identity and it cannot be captured retroactively once state changes.

## 5. Replicate across units before generalising

One failing board supports "this board is defective". Three identical boards support "this batch has a property". The experiments worth running, and the conclusions worth publishing, are different in the two worlds. Where a conclusion matters, unit count is part of the method, not a detail.

## 6. Maintain the elimination list as a first-class artifact

Seven domains were eliminated, each tied to a specific test. That list is what makes the final fault domain a defensible conclusion instead of an opinion, and it is what tells the next engineer exactly where not to re-investigate. In validation work, the eliminated set is deliverable-grade information.

## 7. Stop at the boundary your instruments can defend

The remaining discrimination (bus capture, control board, flash substitution) needs equipment and a control unit this study did not have. The report claims a fault domain, documents the next experiments, and stops. An investigation that claims one step more than its evidence teaches reviewers to discount all of its steps. Restraint is a technical quality, not a rhetorical one.

## 8. Keep the earlier, wronger document

A previous single-board investigation of the same fault class reached a tidier conclusion that this study could not reproduce. That document has been corrected by its author to record the unresolved outcome, and the post-mortem in this repository states the disagreement openly. Engineering records that show conclusions being revised under better data are worth more, to an employer and to the engineer, than a record that pretends to have been right the first time.
