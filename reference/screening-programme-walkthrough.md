# Reference walkthrough — building the population screening plugin

This is a project-specific brief for `care_screening`, the screening-programme plugin described
in issue #25. It turns the idea into an implementation boundary without adding screening logic
to `care` or `care_fe`.

## The brief

> Programme managers define screening programmes and outreach camps. Staff enrol people into a
> programme, record screening results, and assign a follow-up disposition. The system creates
> recalls for people who need repeat screening or clinical review, and shows due recalls to the
> responsible facility team. Patients can see their own recall instructions in the patient portal,
> but cannot see another person's screening record.

The first release should support configurable programmes, camp sessions, an enrolment register,
screening observations, and recall tracking. It should not become a general laboratory,
appointment, or disease-surveillance module.

## Product decisions

| Question | Initial decision |
| --- | --- |
| Plugin names | `care_screening` and `care_screening_fe` |
| Settings prefix | `SCREENING_` |
| i18n prefix | `screening__` |
| Patient access | Yes, through OTP-scoped read-only endpoints |
| Third-party services | None in the first release |
| Record states | Enrolled → screened → follow-up due → closed; authorised staff may move records forward, while facility managers may reopen a closed recall |
| Primary UI | A programme route, facility actions, and a recall worklist |
| Core changes | None expected; use plugin models, `meta`, signals, and manifest routes |

These are defaults for scaffolding, not a substitute for confirming programme policy before
production implementation.

## Where the feature lives

| Requirement | Where it goes | Core cost |
| --- | --- | --- |
| Programme definition, camp, enrolment, observation, and recall | New plugin models inheriting CARE's base model | 0 |
| Programme-specific result fields | JSON configuration on plugin models | 0 |
| Optional screening marker on a core record | Namespaced `meta["screening"]` data | 0 |
| Automatic recall creation after an abnormal result | Plugin service or signal | 0 |
| Staff and patient APIs | Viewsets under `/api/care_screening/` and `/api/care_screening/otp/` | 0 |
| Programme dashboard and recall worklist | `manifest.routes` | 0 |
| Quick action on a facility page | Existing `FacilityHomeActions` extension point | 0 |
| Navigation entry | `manifest.navItems` with a `screening__` label | 0 |

Do not add screening columns to core models or call the plugin from a core viewset. If a later
workflow needs a new attachment point, add one generic extension point only after checking the
catalog in `care-extension-points`.

## Backend shape

```text
care_screening/care_screening/
├── apps.py
├── settings.py
├── urls.py
├── models/
│   ├── programme.py
│   ├── camp.py
│   ├── enrolment.py
│   ├── observation.py
│   └── recall.py
├── serializers/
├── viewsets/
├── services/
├── tasks/
└── tests/
```

Suggested relationships:

- `ScreeningProgramme` owns the programme name, target population, active dates, and configurable
  screening measurements.
- `ScreeningCamp` belongs to a programme and facility and records its date, team, and status.
- `ScreeningEnrolment` references the programme, facility, and core patient, and owns the lifecycle
  state.
- `ScreeningObservation` stores submitted values and a normalised outcome (`normal`, `abnormal`,
  or `inconclusive`).
- `ScreeningRecall` references an enrolment, due date, reason, assignee, and completion status.

All staff querysets must be facility-scoped. Patient endpoints must resolve the patient from the
OTP identity and must never accept a patient identifier from the request body as the authorisation
source.

## API surface

All routes are mounted by core at `/api/care_screening/`.

| Method | Path | Purpose |
| --- | --- | --- |
| GET / POST | `/programmes/` | List or create programmes for authorised staff |
| GET / PATCH | `/programmes/{id}/` | Read or update programme configuration |
| GET / POST | `/camps/` | List or create camp sessions |
| GET / POST | `/enrolments/` | Search or enrol a person in a programme |
| GET / POST | `/enrolments/{id}/observations/` | Read or submit screening observations |
| GET / POST | `/recalls/` | List or create recalls for staff worklists |
| POST | `/recalls/{id}/complete/` | Record follow-up completion |
| GET | `/otp/recalls/` | List recalls belonging to the OTP-authenticated patient |
| GET | `/otp/recalls/{id}/` | Read a patient's recall instructions |
| GET | `/config/` and `/otp/config/` | Return client-safe feature configuration |

Use DRF's standard `detail` and validation error responses. Keep result interpretation and recall
creation in backend services so programme rules cannot be bypassed by a custom client.

## Frontend shape

```text
care_screening_fe/src/
├── manifest.tsx
├── pages/
│   ├── ProgrammeDashboard.tsx
│   ├── CampRegister.tsx
│   └── RecallWorklist.tsx
├── components/
│   ├── FacilityHomeActions.tsx
│   ├── ProgrammeSummary.tsx
│   └── RecallStatusBadge.tsx
└── utils/api.ts
```

Every page and extension component is lazy-loaded from the manifest and wrapped in
`care-screening-container`. User-facing text belongs in `public/locale/en.json` under
`screening__`; the plugin must not edit host locale files.

The staff route can expose filters for facility, programme, due date, and recall status. The OTP
route should expose only clear patient instructions and the next action, not staff notes or the
full programme register.

## Verification bar

- No `care` or `care_fe` file mentions `care_screening`.
- Staff cannot read or mutate records outside their permitted facilities.
- OTP callers see only their own recalls.
- An abnormal observation creates exactly one open recall, even if the request is retried.
- Closing a recall records who completed it and when; reopening is permission-checked.
- `npm run build` succeeds and the generated remote uses `Bearer` authentication.
- Backend tests cover state transitions, facility scoping, OTP scoping, and idempotent recall creation.
