# go-miele: the design

Research into Miele's Wi-Fi connected appliances, a washing machine and a tumble dryer in
view, done on 2026-11-15 from public sources, to decide what a Go module for them would
be. Nothing is built and nothing here was run against an appliance. The decisions at the
end are open.

In short: a Miele appliance serves an encrypted HTTP interface on the LAN, and a client
that holds the appliance's group key can read it there with no cloud. The keys are the
owner's to make, but only on an appliance that has been reset, and the open
implementation's author expects it to exclude the Miele app. Miele's supported interface
is a cloud API, mature, whose availability in the United States is not confirmed.

## The device

A Wi-Fi Miele appliance joins the home's network directly and is added to an account in
the Miele app. Older appliances reach the network through a gateway, the XGW 3000. What
the appliance sends to Miele's servers is not documented.

## Ways in

### The appliance's own HTTP interface (local)

From [MieleRESTServer](https://github.com/akappner/MieleRESTServer), its README and its
`MieleCrypto.py`:

- HTTP on the appliance, resources under `/Devices/`, `/Security/`, `/WLAN`.
- Every request is signed, `Authorization: MieleH256 <GroupID>:<signature>`, an
  HMAC-SHA256 under the group key over the method, the path, the headers and the body.
  Bodies are AES-CBC under the same key.
- The group id and key are random values the owner generates and uploads to the
  appliance at commissioning. An appliance already set up "will not accept a new
  configuration without a reset".
- "Some devices appear to come from the factory in a locked state": the upload then
  fails with a 403 until the appliance has been added to the official app and removed
  again. The README names one dryer with it.
- Read: the appliance's identity and state. Do: start, and only that: "The server is not
  yet capable of programming the device, but it can start the device if the programming
  is done locally", and some appliances want a setting changed on their panel first.
- The author's view, in the project's thread on the Home Assistant community forum,
  which its README links: local keys and the cloud are a choice of one.

### Miele's cloud API (official)

- `https://api.mcs3.miele.com/v1`: the devices, each one's identity, state and
  available actions ([general concept](https://developer.miele.com/docs/general-concept)).
- OAuth by authorization code, with credentials requested from Miele's developer portal.
  [The authorization page](https://developer.miele.com/docs/authorization) names two
  providers: a new one, listed for European countries and mainland China, and a legacy
  one, "still active for now", recommended for the countries the new one lacks, none of
  which it names.
- Changes arrive as server-sent events.
- For laundry: start, stop, pause, power on and off, where the appliance reports them
  available. Starting needs the appliance's Mobile Start turned on at the appliance for
  that load ([Home Assistant's page](https://www.home-assistant.io/integrations/miele/)).

### Matter

None found on Miele laundry.

## Two ways in, compared

| | The appliance's HTTP interface | The cloud API |
|---|---|---|
| Supported by Miele | no | yes |
| Local at run time | yes | no |
| Needs | the appliance reset and commissioned with one's own keys | developer credentials, OAuth consent |
| Gives | identity and state; start of a cycle set at the panel | state, events, start, stop, pause, power |
| Cost | the Miele app and cloud for that appliance, by the author's account | a cloud in the path; coverage of the United States not confirmed |

## What this module would be

Not decided. The options, most local first:

1. **A client of the appliance's HTTP interface**, a port of MieleRESTServer's signing
   and encryption, which are small and fully in its source, with the commissioning steps.
   The open part is decoding the state of the models in view, and whether losing the app
   is acceptable. Trying the Python tool against the appliances first would answer both.
2. **A client of the cloud API**: OAuth and an event stream. Small, once credentials are
   had for the account's country.

No Go implementation of the local protocol was found; of the cloud API,
[miele-go](https://github.com/IngmarStein/miele-go), archived.

## Open questions

- The model numbers of the washer and the dryer.
- Whether resetting them and leaving the Miele app is acceptable.
- Whether watching a cycle is enough, or starting one is wanted: locally a program
  cannot be chosen.
- Whether Miele's developer portal gives credentials for the account's country.

## Not verified

- Nothing above was observed on an appliance.
- That local keys and the cloud exclude each other is the author's expectation; no test
  either way was found.
- Which state fields a washer and a dryer report locally: the attribute tree is only
  partly decoded.
- Any limit on the cloud API's requests.
- Reports for North American models on the local interface.

## Sources

- [MieleRESTServer](https://github.com/akappner/MieleRESTServer): the local protocol,
  commissioning, what works and on which appliances.
- Miele: [general concept](https://developer.miele.com/docs/general-concept),
  [authorization](https://developer.miele.com/docs/authorization),
  [eventing](https://developer.miele.com/docs/eventing).
- [Home Assistant, Miele](https://www.home-assistant.io/integrations/miele/): the cloud
  API in use, Mobile Start.
- [pymiele](https://github.com/astrandb/pymiele): the cloud API's hosts and actions.
- [miele-go](https://github.com/IngmarStein/miele-go): a Go client of the cloud API,
  archived.
