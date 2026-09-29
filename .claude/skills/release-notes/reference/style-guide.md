# What's new style guide

Anchor on entries `0005` to `0008`. Entry `0001` has a stray `##` where a `###` belongs and older French wording; do not copy it.

## File skeleton

No frontmatter. UTF-8, LF line endings, one trailing newline, a blank line between blocks. Only `#`, `##` and `###` headings. Omit any section or subsection that would be empty.

```
# <H1>

<intro line>

## <New features>

### <Short noun phrase>

One or two sentences, or `-` bullets when there are two or more sub-points.

## <Improvements>

### ...

## <Academics>

### ...

### <Fixes for educators>

- **Lead-in** — rest of the sentence.

## <Fixes>

- **Lead-in** — rest of the sentence.

---

<footer line>
```

## Fixed strings

| | en | es | fr |
| --- | --- | --- | --- |
| H1 | `# What's new in <Month> <YYYY>` | `# Novedades de <mes> de <YYYY>` | `# Nouveautés de <mois> <YYYY>` (elide to `d'` before avril, août, octobre: `Nouveautés d'août 2026`) |
| Intro | `A roundup of recent improvements across the platform — new features, polish, and fixes.` | `Un resumen de las mejoras recientes en la plataforma: nuevas funcionalidades, retoques y correcciones.` | `Un tour d'horizon des améliorations récentes sur la plateforme — nouvelles fonctionnalités, retouches et corrections.` |
| New features | `## New features` | `## Nuevas funcionalidades` | `## Nouvelles fonctionnalités` |
| Improvements | `## Improvements` | `## Mejoras` | `## Améliorations` |
| Academics | `## Academics` | `## Académico` | `## Académique` |
| Academic fixes | `### Fixes for educators` | `### Correcciones para docentes` | `### Corrections pour les enseignants` |
| Fixes | `## Fixes` | `## Correcciones` | `## Corrections` |
| Footer | `Have feedback? Reach us through the in-app help.` | `¿Tienes comentarios? Escríbenos desde la ayuda integrada.` | `Des retours ? Contactez-nous via l'aide intégrée.` |

Section order: New features, Improvements, Academics, Fixes. Academics sits with the prose sections; Fixes stays the flat list at the end.

## Months

| `%m` | en | es | fr |
| --- | --- | --- | --- |
| 01 | January | enero | janvier |
| 02 | February | febrero | février |
| 03 | March | marzo | mars |
| 04 | April | abril | d'avril |
| 05 | May | mayo | mai |
| 06 | June | junio | juin |
| 07 | July | julio | juillet |
| 08 | August | agosto | d'août |
| 09 | September | septiembre | septembre |
| 10 | October | octubre | d'octobre |
| 11 | November | noviembre | novembre |
| 12 | December | diciembre | décembre |

French H1: `Nouveautés de septembre 2026`, `Nouveautés d'août 2026`. Spanish H1: `Novedades de septiembre de 2026`.

## Typography

- **en**: straight double quotes, spaced em dash ` — ` in Fixes bullets.
- **es**: «guillemets» without inner spaces, `¿…?` and `¡…!`. Fixes bullets use a colon instead of the dash: `- **La plantilla**: ya no …`.
- **fr**: « guillemets » with inner spaces, a space before `?`, `:` and `;` (`Des retours ?`), spaced em dash ` — ` in Fixes bullets.
- Straight apostrophes (`'`) in every language.
- Bold one key noun phrase per note, never a whole sentence.

## Glossary

Keep these product nouns untranslated in all languages: Libertify, Playbook(s), Knowledge Hub, BrandKit / brandkit, LTI, SCORM, Moodle, LMS, Quick Recall, Quiz, Trivia, Podcast, Study Chatbot, Key points, Standouts, Hey Genius, admin, media-player, sales (app names), storyboard.

| en | es | fr |
| --- | --- | --- |
| add-on | complemento | module |
| viewers | espectadores | spectateurs |
| teachers | docentes (collective: profesorado) | enseignants |
| students | alumnos | élèves |
| educators | docentes | enseignants |
| upload (noun / verb) | subida / subir | téléversement / téléverser |
| flow | flujo | parcours |
| modal | el modal | la modale |
| highlights | resaltados | surlignages |
| Bookmarks tab | pestaña Marcadores | onglet Favoris |
| onboarding | onboarding | onboarding |
| questionnaire | cuestionario | questionnaire |
| experience v2 / new experience | experiencia v2 / nueva experiencia | expérience v2 / nouvelle expérience |
| storyboard editor | editor de storyboard | éditeur de storyboard |
| video editor | editor de vídeo | éditeur vidéo |
| Share modal | modal Compartir | modale Partager |
| call-to-action (CTA) | llamada a la acción | appel à l'action |
| spinner / loading state | spinner / estado de carga | indicateur de chargement |
| toast | notificación | notification |
| two-factor authentication | autenticación en dos pasos | authentification à deux facteurs |
| tab strip | barra de pestañas | barre d'onglets |
| profile page | página de perfil | page de profil |

## Do and don't

| Don't (commit subject) | Do (customer note) |
| --- | --- |
| `Fix PdfLoader url-change race; self-host the pdf.js worker` | `**PDF loading** no longer fails when a document's link refreshes, and the PDF engine now ships with the app for more reliable loading.` |
| `Wire logo for skema` | Drop it. Customer-specific work is never mentioned. |
| `Update Types from Backend, wire remaining settings, move settings from Video to General` | `**General settings** such as language now live in their own card instead of inside the Video add-on.` (the type sync is internal and dropped) |
| `Fix the failing CI test jobs: storybook flake forensics, tracking e2e unblocked` | Drop it. Not customer-facing. |
| `[MED-7163] Never save speech for chapters` + `[MED-7163] Revert clearing the chapter's speech` | Drop both. A change and its revert in the same release cancel out. |
