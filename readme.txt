This is an adaptation of the BLE_HeartRateFreeRTOS example to use the P2PServer.

The main sites of action are app_ble.c, p2p_server_app.c/.h, app_entry.c, app_conf.h, and svc_ctl.c

APPE_Init is called from main; this calls APPE_SysUserEvtRx when complete, which calls APP_BLE_Init.
APP_BLE_Init calls the Service registering function SVCCTL_Init, which will register all services.
These services are weakly defined as empty functions unless we add and link in our C files with
our services; we can either delete the HR and DIS service used in the heartrate example (that we 
don't need or want) or we can go into svc_ctl.c and remove the DIS_Init() and HRS_Init calls from SVCCTL_SvcInit().

SVCCTL_SvcInit calls P2PS_STM_Init(), which is in Middlewares->STM32_WPAN->p2p_stm.c.  This init
function initializes and registers the services we care about with the service handler; p2p_stm.c has
a couple additional calls I've added (P2PS_STM_App_Update_Int8()/Int16()) which just conveniently
extend the default call that sends a char value.  For this example, it doesn't matter as we only
send chars, but it's good to remember to fix this.
 
Back in app_ble.c, we've modified APP_BLE_Init to call the P2P_APP_Init() function; we've also added
P2PS_APP_Notification calls for bluetooth events in SVCCTL_App_Notification (these are 
connect/disconnect events, not data-- data update events are routed to P2PS_STM_App_Notification 
using the service controller.  P2PS_APP_Notification just allows us to reinitialize context when 
the app disconnects/reconnects). 

We've also rewritten the APP_BLE_Key_ButtonX_Action() functions, which are called from EXTI interrupts
on button presses, so that only one of them is active and calls P2P_APP_SW1_Button_Action (set a flag).

Finally in app_ble, we've changed the name of service from 'HRSTM' to 'DRAMSAY' in the two places
it occurs, when setting up the registration of the device.

p2p_server_app.c has been rewritten significantly so that the init function creates a thread function
P2PProcess that calls the P2PS_Send_Notification function when its Flag is set; the Button push now sets the flag.

The receive events in p2p_server_app.c work as intended without modification, as long as they are
registered with the svc_ctl (which they are).

Lastly, we need to modify app_conf.h-- we turn off SMPS, we turn on several debugging flags so that
our system can be breakpointed, we add some config typedefs for our P2P thread, and we turn on LED
and button support, which are turned off by default in this example.

Now it works just like the standard P2P_Server example, but we've contextualized it in FreeRTOS.


A final note: when importing examples and modifying them, CubeIDE does some weird things; it saves
all the files in the base directory, but any modifications you make get put in a secondary folder
within that directory called 'STM32CubeIDE'.  I've merged these together to make this example
readable.  I don't know why project management and state management for this code seems to be so
difficult to get right with CubeIDE/Eclipse, but here we are.  It may be easiest to load in the
example and copy over the modified files by hand.