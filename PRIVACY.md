# Privacy Policy: Cable Ampacity MCP connector

Operator: CDK AI System. Contact: chedy028@cdkaisystem.com. Effective date: November 21, 2026.

This policy covers the cable ampacity connector at https://cable-ampacity-mcp.onrender.com/mcp during the open beta. The connector is hosted on Render. It does not cover any other service.

## 1. No sign-in

During the open beta the connector has no sign-up, no user profile, and no password. Callers do not sign in, and the connector does not require an API key. This service's application code does not set a cookie. Each request is handled on its own.

## 2. What a call sends

When an AI assistant calls a tool, it sends the tool name and the inputs. The inputs are engineering values such as cable size, material, temperatures, installation geometry, and load. The `site_ambient` tool also receives a latitude, a longitude, and a depth. A cable rating can also include a site location, used to compare an ambient temperature with measured station data.

The service also sees the network address of the request and ordinary request headers.

## 3. Network address

The server reads the caller address to apply a rate limit and to count requests. On this host that address comes from the `X-Forwarded-For` header. The server uses the caller entry in that header. If the header is missing or too short, it uses the connection address. If that is missing too, it uses the label `unknown`.

An address outside the ranges below is limited to 60 requests in 60 seconds. Addresses in `160.79.104.0/21` share one limit of 600 requests in 60 seconds. Addresses in `153.61.208.0/20` share a separate limit of 600 requests in 60 seconds. Claude's connector traffic arrives from those two ranges.

The rate limit is kept in server memory. What is stored is the counter key and the times of requests inside the current 60 seconds. The counter key is the address, or the network for the two ranges above. The tool name and the inputs are not in this list. Times older than 60 seconds are dropped. A restart clears the list. If the list grows past 4096 entries, the least recently used entries are dropped.

The server also keeps a request count in memory. A row stores the individual network address, a label, the path, and a count. For this connector the path is `/mcp`. During the open beta no API key is required. A request that sends none is labeled `anonymous`. If a request includes an API key, the label is the first 8 characters of the SHA-256 hash of that key. The key itself is not stored. The tool name and the inputs are not in this count. For Claude's ranges the rate-limit counter is the network, and the request count still stores the individual address. These rows are not written to the database and are not sent to an analytics service. A restart clears them. If the list grows past 4096 rows, the least recently used rows are dropped.

The address is not stored in a saved calculation run.

## 4. Saved calculation runs

Six tools save each successful calculation: `compute_ampacity`, `compute_ampacity_iec`, `compute_ampacity_neher_mcgrath`, `rate_cable`, `select_cable`, and `run_study`. `select_cable` and `run_study` can save one run for each cable they calculate. A failed calculation is not saved. The other fourteen tools on this server do not save a run.

A saved run has a random run ID, a short question, the normalized inputs (including values the calculation filled in), intermediate values, results, warnings, and a generated report. Inputs the calculation does not recognize are rejected and are not saved. The report does not include the text of the standards. The run does not include the network address.

The short question names the calculation. For `compute_ampacity` it includes the conductor size, material, and insulation. For `compute_ampacity_iec` and `compute_ampacity_neher_mcgrath` it includes the conductor size. For `rate_cable`, `select_cable`, and `run_study` it names the tool and the method.

A site location sent only for an ambient comparison is not written into the saved run. `site_ambient` reads a data table shipped with the server. It does not save the location, and it does not open a network connection.

## 5. Who can read a saved run

Anyone who has a run ID can fetch that run with `get_run`. No sign-in is required. The reply includes the calculation outputs, the intermediate values, and the ledger events. Those values describe the calculation. The events include the short question. `get_run` does not return the stored report. There is no tool that lists every run ID.

Do not put personal or confidential information in your inputs. A run ID is not a password. Share it only with people who may see that calculation.

## 6. How long runs are kept

A saved run is deleted after 7 days. The server checks about once an hour, so a run can remain a little longer than 7 days until that check runs. A run is deleted sooner when the service is redeployed or restarted, because the database file sits on the server's temporary disk. This service does not keep backups of saved runs.

The in-memory rate limit and request counts are cleared when the server restarts.

## 7. Access logs

The web server writes an access log line for each request. The line has the connection address, the request line, and the status code. It does not include tool inputs. These logs are retained by Render per its standard retention period. The application does not log tool inputs.

## 8. Selling, advertising, and training

This connector does not sell data. It does not use data for advertising. It does not use inputs to train models. It does not send tool inputs to any other service. The connector is hosted by Render, in Render's Oregon region. A request passes through Render on the way to the application.

The AI assistant that calls the connector, such as Claude, is run by its own provider and has its own privacy policy. This policy does not describe what that provider stores.

## 9. Changes

This policy may be updated. The effective date will change when that happens.

## 10. Contact

Questions: chedy028@cdkaisystem.com.

Postal address: 2652 SW 9th Terrace, Lee's Summit, MO 64081.
