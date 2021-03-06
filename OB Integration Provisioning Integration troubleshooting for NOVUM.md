# OB Integration Provisioning Integration troubleshooting for NOVUM
> Technical assessment

OB integration & gOB Task:
: NOVUM: gOB ElasticSearch Dashboards and WoW for gOB


# Index

1. References & Related Inf
2. Objective, assumptions & current situation
2.1 WHAT?
2.2 WHY?
2.3 HOW?
3. Technical assessment
4. gOB current troubleshooting solution
4.1 gOB Call and SMS CDRs
4.2 gOB run logs
4.3 gOB fault alarms & performance alarms
4.4 NGIN provided tools
5. OB integrations current troubleshooting tools
5.1 Splunk
5.2 Voipmonitor
6. Short-term troubleshooting solution for NOVUM
7. Mid/long-term troubleshooting solution evolution for NOVUM
8. Conclusions

# 1. References & Related Info

# 2. Objective, assumptions & current situation


### 2.1 WHAT?
In the case of TU the provisioning flow was managed e2e from gBE. Therefore it used to be SEEN Team who owned the control and troubleshooting of this flow.
In the case of NOVUM the provisioning flow changes radically, therefore the way we work must change. 
Regaring OB Integration share the provisioning flow starts when ObProv component dispach the ACTIVATE service event towards rSDP and finish when gOB recieves the startCall and startSMSNotifications events.
This means that we should be able to give mainly support about gOB provisioning status, but it shall be discused if we may also give support regaring SDP and OB provisioning status as a technical reference team for the OBs.

~~~~
Sobre flujos de provision, ¿Mantendremos responsabilidad e2e de los flujos de provision como legacy de SEEN?
~~~~



## 4. WHY?
This document is intended to analyse current troubleshooting solution of OB-related arround provisinioning how to improve it and evolve it according to NOVUM requirements.
 
Due to OB Integration is the main technical focalpoint team for the OBs regarding IPComms integration, we should be able give support outside NOVUM BE and SDP integration.
~~~
+ Que sucede con SDP, es parte importante de la integración pero es un componente cuya responsabilidad es de la OB, Nuestro punto de observación sería ObProv Component.  ¿Debemos crear nosotros las herramientas de troubleShooting en ObProv,  tal como hacemos en los servers OREJAS de gBE?
~~~



## 5. HOW?

As well as It’s required to define what would be the way to troubleshoot issues in gOB, and in the OB,  for NOVUM since Splunk is not going to be available we would have to replicate dashboards.
The place and/or tool where all the info generated by OB platforms like mSDP error log events, alarms should be available for the organization must be defined and agreed for NOVUM. 

An idea could be reuse TU TRIPAS server where accessing via SSH we can upload logs and indexing them by fluentd to ElasticSearch Core and visualize them throgh Kibana in case of NOVUM.

## 6. Technical assessment

Provisioning current troubleshooting solution

### Provisioning flow
To have a look to the end to end please go to [provisioning flow]( http://www.websequencediagrams.com/files/render?link=VPDZDsby5MzNYUUQltVe) and component source type PoV

! [Alt Provisioning Flow](https://github.com/augustodevesa/OBIN_Common_repository/blob/master/Troubleshooting_Splunk_v3.jpg)

Flow has 3 asynchronous stages. Best way is to correlate by MSISDN and look for BI events:

POST users/register, from App, reaching  API. Logs GC1001 or GC1002 event in BI
Token: unknown:$hash_misdn:$random
gBE checks that requester MSISDN belongs to OB and sends OTAC (preventing abuse)

POST /users/validate, from App, reaching  API. Logs GC1003 or GC1004 + optionally GC1005 or GC1006 event in BI
Token: unknown:$hash_misdn:$random
gBE validates PIN and coordinates provision in other 3 platforms (TU Core, gOB and OB). User moved to PENDING. Error if PIN is not validated or abused. Rollback of already provisioned systems if failure.

UNICA Event Act0001, from OB, reaching NOTIFICATIONS\_CONTROL. Logs GC1005 or GC1006 event in BI 
Token: unknown:$hash_misdn:$random
OB notifies result of activation. If success user is moved to ACTIVE. If failure, rollback and user is moved to DELETED. SMS sent with result, Reminder scheduled


| BI Event | Meaning | Consequence |

GC1001:
> User belongs to OB, after UNICA User Context request 
> PIN is generated, stored in gBE DB and sent to user by SMS API 2xx response to Register request |

GC1002
> User does not belong to OB (or Not Elegible in case OB supports early checking)
> 4xx response to Register request. User informed that is not elegible for TU Go |

GC1003
> PIN validated
> User is validated and provision starts in this order: TU Core, gOB and OB. If no error in provision and OB answers with Activation Pending: user is moved to PENDING status and 2xx response to validate |

GC1004 
> PIN not validated 
> 4xx response to validate and provision is aborted |

GC1005 
> Activation Completed 
> User is moved to ACTIVE status. SMS sent informing the user. Reminder SMS enqueued |

GC1006 
> Activation Failed 
> Failure reason logged in BI event. Provision rolled back. 4xx/5xx to validate if activation failed before reaching PENDING status. SMS sent informing the user |


Typical sequences and meaning (+: 1 or more, *: 0 or more):
-----

+ GC1002: Registration request by user from another OB (or not elegible if early checking)
+ GC1001: User requested PIN but never tried to validate
+ GC1001 > GC1004+: PIN tried but never validated
+ GC1001 > GC1004* > GC1003: PIN validated and Pending Activation 
+ GC1001 > GC1004* > GC1003 > GC1006: Activation failed after PIN validation (see Reason field)
+ GC1001 > GC1004* > GC1003 > GC1005: Activation completed

#### Events 

**EV\_US\_Act0001 (success)**
Success result of a request initiated by gBE

1. gBE (Orejas) receive event from OB (/ob/$ob\_id/events/v1), validates event and queues it to Manos
2.  Updates user status in gBE DB to ACTIVE
3. Send SMS to user, UNICA:  POST /services/SendMessage\_1\_1
4. Logs BI GC1005
5. Update user status in TU Core

Example
~~~~
{ "phone_number":"4477966xxxx",
  "event_type":"Act0001",   "success":true,   "event_name":"Process Concluded",
  "timestamp":"2014-10-17T23:57:42.000+01:00",   "ob":"23411",
  "_token":"unknown:2860308605:126", // unknown:$hash_msisdn:$random
  "action":"Activation“ , // Activation | Deactivation
  "service_id":"UKgConn", "transaction_id":"44a9a64f-501f-482c-b97a-ba60ec36b8de"}
~~~~


**EV\_US\_Act0001 (failure)**
Failure result of a request initiated by gBE

1. gBE (Orejas) receive event from OB (/ob/$ob\_id/events/v1), validates event and queues it to Manos
2. Logs BI GC1006
3. Rollback TU Core. BES: DoesUserExist, DeleteCommunicationRecords, GetUser,  GetUserDevices, RemoveUserDevice, RemoveUserPhone, UnregisterPlan, UnregisterUser
4. Rollback gOB: UNICA: DELETE /callnotification/v2/subscriptions, DELETE /smsnotification/v2/subscriptions
5. Updates user status in gBE DB to DELETED
6. Send SMS to user with text mapped from failure\_reason, UNICA:  POST /services/SendMessage\_1\_1

Example
~~~~
 { "phone_number":"4479553xxxxx",
  "event_type":"Act0001",   "success":false,   "event_name":"Process Concluded",
  "timestamp":"2014-10-17T23:34:49.000+01:00",   "ob":"23411",
  "failure_reason":"UK0015: Backend technical error: ABS error - 990\/gConnect subscriber is currently barred", // Filled by OB. Format agreed with TU Go: <OB>xxxx. This code is used to map the failure reason with the text of SMS to be sent to users
  "_token":"unknown:2860308605:969",
  "action":"Activation", // Activation | Deactivation
  "service_id":"UKgConn",   "transaction_id":"5fee5f0c-3694-424f-8cd9-36c566ec3e41"}

~~~~

**EV_US_Act0002**
Notification from OB about a process initiated in OB

1. gBE (Orejas) receive event from OB (/ob/$ob_id/events/v1), validates event and queues it to Manos
2. if action: Suspended (and user is PENDING or ACTIVE):
2.1.  Logs BI GC1010
2.2. Deletes gOB subscriptions: UNICA: DELETE /callnotification/v2/subscriptions/, DELETE /smsnotification/v2/subscriptions/
2.3. Resets TU Core password: BES: UpdateUser
2.4. Updates user status in gBE to SUSPENDED

3. if action: Activation (and user is SUSPENDED)
3.1. Logs BI GC1012
3.2. Creates gOB subscriptions: UNICA: POST /callnotification/v2/subscriptions/, POST /smsnotification/v2/subscriptions/
3.3 Updates user status in gBE to ACTIVE

4. if action: Dectivation (and user is PENDING, ACTIVE or SUSPENDED)
4.1. Logs BI GC1007
4.2. Unregister user in TU Core: BES: DoesUserExist, DeleteCommunicationRecords, GetUser, 4.3. GetUserDevices, RemoveUserDevice, RemoveUserPhone, UnregisterPlan, UnregisterUser
4.5 Deletes gOB subscriptions: UNICA: DELETE /callnotification/v2/subscriptions/, DELETE /smsnotification/v2/subscriptions/
4.6 Updates user status in gBE to DELETED

Example:
~~~~
{ "phone_number":"44785610xxxx",
  "action":"Suspended", // Activation | Deactivation | Suspended
  "reason":"Initial gConnect bar added", // Filled by OB
  "event_type":"Act0002",   "service_id":"UKgConn",   "event_name":"Status Updated",
  "timestamp":"2014-10-17T20:26:48.000+01:00",   "_token":"unknown:2860308605:664",   "ob":"23411",
  "failure_reason":"Error while provisioning. Unknown reason"} // Irrelevant, always this (unless in UK)
~~~~


### Provisioning fault & performance alarms

**Useful dashboards: app/tugo/provision_stats**


% of use cases for requests and users:
----
+ User from another OB, no OTAC: Certain % expected. Really it shows that User Context API has not reported that user is OK for TU Go. In UK reason is that user does not belong to OB but other OBs may perform eligibility check here. If it increases it may show that UserContext is failing
+ User did not try OTAC validation: Small % expected. If it increase it may show problems sending SMS with PIN code.	
+ User failed OTAC validation: Very small % expected
+ User passed OTAC but failed activation: Certain % is expected as Users not Eligible are rejected at this stage. Take a look to causes of failure reason
+ User passed OTAC but pending activation: Users left in PENDING status. If it increase it may be due to OB not sending Act0001 or TU Go not processing it correctly.
+ User completed activation: Success rate 


Causes for failure reason:
----
+ Number of occurrences and number of different MSISDNs affected. 
+ For non-temporary errors (such as user Not Eligible) if those 2 values differ a lot it shows that UX is not working well, as users are retrying when they shouldn’t
+ Failure reasons containing OBxxxx codes (e.g. UK0014) are due to OB. Others may be due to our systems.


Distribution of Delays:
-----
+ Delay for OTAC validation (GC1001 to GC1003): How long it takes for users to get the PIN code and validate it
+ Delay for activation after validation (GC1003 to GC1005): How long it takes the provisioning process for users activated. May delay it is due to OB usually.
+ Delay for synch activation failure after validation (GC1003 to GC1006 in Orejas): How long it takes provisioning process until users are synchronously rejected by OB (usually because user is not eligible)
+ Delay for asynch activation failure after validation (GC1003 to GC1006 in Manos): How long it takes provisioning process until users are asynchronously rejected by OB (by means of Act001)



#### Splunk

Currently Splunk is the tool for troubleshooting e2e all TUGO Flows. In the case of Provisioning flow there are several dashboards, alerts and reports being use to monitor, and tackle provisioning incidence in all OBs where TU has been implemented.

Such is the strenght of the tool that is quite often the NOC detects an incidence not related to the TU but with the OB network early in advance before the OB is able to detect it. 

This are the examples of the dashboards currently used.
+ [Troubleshooting_Splunk_v3](https://docs.google.com/presentation/d/1Zw3oi2DK-KgRUzCSwi6dmA3pIrgh1EBPtwVdueS6kTE/edit#slide=id.p28)
+ [Dashboards: Splunk Dashboards of MIA](https://10.253.1.11/en-US/app/search/dashboards)
+ [TUGo KPI](https://10.253.1.11/en-US/app/tugo/basic_monitoring?earliest=-15m&latest=now)
+ [TUGo Subscription KPI](https://10.253.1.11/en-US/app/tugo/subscriptions?earliest=-7d%40h&latest=now)
+ [TUGo Provisioning Monitoring](https://10.253.1.11/en-US/app/tugo/provision_argentina)
+ [TUGo Provisioning Form](https://10.253.1.11/en-US/app/tugo/provision_stats?earliest=-4d%40d&latest=%40d&form.rel_time=-0h&form.ob_selector=72410)
+ [Trouble Shooting](https://10.253.1.11/en-US/app/tugo/tu_go_comm_flows)
+ [TUGo Diagnostics/Alerts](https://10.253.1.11/en-US/app/tugo/alerts) 
+ [TUGo Diagnostics/Dashboards](https://10.253.1.11/en-US/app/tugo/dashboards)



NOVUM Provisioning Flow 
====================

The provisioning flow specs is defined on the following [document](https://docs.google.com/document/d/1dkvbBnjqc0by130jircMHrWzpfqesGe9YM45lyczzBc/edit#)

On the scope of OB integration team the significant share is the ObProv component behavior as Service interface with rSDP/mSDP/OB provisioning. 

In order to track the events ObProv is going to send envents to BIEventsService and implement logstash to integrate them into ElasticSearchCore and therefore Kibana /Graphana UI.



Troubleshooting solution for NOVUM
============================


The short-term troubleshooting solution for the provisioning flow in NOVUM will be composed by:

### Service CDRs

CDRs integration have to be transparent for the OBs. The CDRs of the new service must keep the same format and respect the same values on the fields that the OB use for their post processing.

Therefore any change CDRs collection and harvesting process would affect the OBs requesting new customizations developments.
 In order to avoid this the NOVUM CDRs have to be accessible to the collections scripts running on TRIPAS.
 
 Moreover, as well as gOB CDRs and Runlogs, service CDRs all hosted on the TRIPAS servers should be included on EFK to accessible through Kibana for trouble shooting purpose.

As wee as for gOB Runlogs & CDRs It’s a MUST for NOVUM to gather this files.

+ Tagging of metadata (source, sourcetype, and host)
 + Configurable buffering
 + Data compression
 + SSL security
 + Use of any available network ports
 + Running scripted inputs locally



Since the OB integration and gOB solution is mostly shared by TU and NOVUM, Splunk can be still used in the short-term to check NOVUM related stuff. For example to check rSDP issues, NGIN issues or any other problem related to flows based on UNICA APIs… which in many cases could be common for TU and NOVUM. However, in the short-term we should be able to use NOVUM metrics and logs available in Grafana and Kibana respectively. 


### Metrics and Logs for NOVUM

As described on the Tech PID: Novum OB Provisioning document the Metrics to monitor are
+ Number of events from rSDP
	+ By action, success and failure_reason
	+ Number of responses returned
+ Number of incoming provision requests from Subscription service
+ Number of events forwarded to SubscriptionsService
	+ By action, permanent/transient error
+ Count of response codes/timeouts to external APIs.
+ Histograms for delays in requests to external components (APIs, DB…)





Metrics for OB Provisioning Service are going to be able via Grafana. Logs for OB Provisioning Service via Kibana.

Below is provided the High level architecture for metrics, monitoring, BI and CDRs in NOVUM: 

[Metrix and Monitoring HLD Arq](https://github.com/augustodevesa/OBIN_Common_repository/blob/master/High%20level%20arch%20for%20metrics%2C%20monitoring%2C%20BI%20and%20CDRs.png)


`2017/05 At this point in time the architecture above is being defined and there are no specific dashboards created for NOVUM B2C in ARG. By the time the service is commercially launched, the required metrics and logs must be available.


[Below an example of Grafana is provided](https://grafana.prd-mia.tuenti.io/dashboard/db/obprovisionservice?from=now-12h&to=now)



[This is an example of Kibana is provided](https://kibana.prd-mia.tuenti.io/app/kibana#/discover?_g=(refreshInterval:(display:Off,pause:!f,value:0),time:(from:now-4h,mode:quick,to:now))&_a=(columns:!(_source),filters:!(('$state':(store:appState),meta:(alias:!n,disabled:!f,index:'logstash-*',key:namespace,negate:!f,value:movistar-ar),query:(match:(namespace:(query:movistar-ar,type:phrase)))),('$state':(store:appState),meta:(alias:!n,disabled:!f,index:'logstash-*',key:level,negate:!f,value:INFO),query:(match:(level:(query:INFO,type:phrase)))),('$state':(store:appState),meta:(alias:!n,disabled:!f,index:'logstash-*',key:serviceName,negate:!f,value:obprovision-service),query:(match:(serviceName:(query:obprovision-service,type:phrase))))),index:'logstash-*',interval:auto,query:(query_string:(analyze_wildcard:!t,query:'*')),sort:!('@timestamp',asc)))



> Kibana is actually providing the web interface where we connect to, but the solution is  composed by a full suite of modules (https://www.elastic.co/products). Kibana is also > known as ELK (Elasticsearch, Logstash & Kibana) or EFK (Elasticsearch, FluentId & Kibana) which are analogous to the solution provided by Splunk. 
Mid/long-term troubleshooting solution evolution for NOVUM

We should be able to get this type of metrics for Novum  (metrics section)

+ Activation typically takes between 10 and 30 seconds. Failures are usually under 10s


+ Compared to other OBs, ratio of activation failures is relatively low, but it’s frequent to see peaks in failures due to transient errors


+ Aggregated results for last 20 days:


result | failure_reason | % 
Activated | - | 96.31
Failed | SVR1000:Generic Server Error | 1.70
Failed | SVC4006: Subscriber Not Elig | 1.58
Failed | 1208 | 0.30
Failed |403-SVC1022-Overlapping subs | 0.10
Failed |SVR1008:Timeout processing r | 0.01

We’ll need to map failure reasons to statuses notification.
Events received from OB (30 days):
+ ActXXXX events refer to Novum/Tugo Service.
+ OBXXXX events refer to whole OB service.

event\_type | event\_name | action | success | failure\_reason |  n\_events | unique_users
OB0003 | User Status Changed | Suspended | 0 | Error while provisioning. Un | 9796 | 9624
Act0001 | Process Concluded | Activation | true | 0 | 9785 | 9739


Responses to Failure at step RegisterObUser (sync response to ActivateService in OB)

ob | reason | n\_responses | %_ob | t
ar | 403-SVC1022-Overlapping subscriptions Already existing. Subscription Identifiers are...  | 117 | 99.2 | 118


Statuses in TU when certain event was received
Some unexpected transitions might happen.

event\_from\_ob | ACTIVE | DELETED | DELETING | PENDING | SUSPENDED
Argentina: Activation Completed | 22 | 4 | |  7789
Argentina: Activation Failed | 11 | 13 | |  951 |  
Argentina: Deactivation Completed





# The scope of gOB on the provisioning flow 

Keep current interfaces available for TU product:
+ Start/Stop Call Notification.
+ Start/Stop SMS Notification.
Add new interface to be able to gather current subscription:
+ Get Call Notification status.
+ Get Sms Notification status.



### BI CDRs 

Old BI CDRs are no longer be needed as BI integration over 4 platform is being done by TU and OB BI teams.


Conciliation Procedure
------------------------------

Due to the changes on the provisioning flow and the different DDBB involved a whole new conciliacion procedure is going to be needed.

If it's posible some of the rules could be reused but must be reviewed as the impact could be absolutely different.



## Conclusions & Next Steps

In the shortterm, we could still use Splunk dashboards and metrics by readapt them to the new provisioning flow. A short project is going to be needed for so. 
The way of working with the NOC could also be reused 

In the mediumterm, the dashboads and metrics from splunk should be moved to kibana / graphana

In long run, the action protocol for the NOC has to be rebuild.

On behalf of Service Managment Ivan Fridman and Rodrigo Da Silve are rebuilding the operations protocol in order to deliver the NOC and internally and end to end mnitoring and supervising view  of the service.




----


