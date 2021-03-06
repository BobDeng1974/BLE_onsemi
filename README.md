
```
int main(void)
{
    uint32_t length;
    uint8_t temp[BUFFER_SIZE];
    /* Initialize the system */
    App_Initialize();

    /* Main application loop:
     * - Run the kernel scheduler
     * - Send notifications for the battery voltage and RSSI values
     * - Refresh the watchdog and wait for an interrupt before continuing */
    while (1)
    {
        Kernel_Schedule();

        if (unhandled_packets != NULL)
        {
            if (UART_FillTXBuffer(unhandled_packets->length,
                                  unhandled_packets->data) !=
                UART_ERRNO_OVERFLOW)
            {
                unhandled_packets = removeNode(unhandled_packets);
            }
        }
        // 蓝牙连接上
        if (ble_env.state == APPM_CONNECTED)
        {
            if (app_env.send_batt_ntf && bass_support_env.enable)
            {
                app_env.send_batt_ntf = 0;
                Batt_LevelUpdateSend(0, app_env.batt_lvl, 0);
            }

            if (cs_env.sentSuccess)
            {
                /* Copy data from the UART RX buffer to the TX buffer */
                // 串口接收数据，保存到temp中，接收数据的长度是length
            	length = UART_EmptyRXBuffer(temp);
                if (length > 0)
                {
                    /* Split buffer into two packets when it's greater than
                     * packet size */
                	// 通过notify发送数据
                    if (length > PACKET_SIZE)
                    {
                        CustomService_SendNotification(ble_env.conidx,
                                                       CS_IDX_TX_VALUE_VAL,
                                                       temp,
                                                       PACKET_SIZE);
                        CustomService_SendNotification(ble_env.conidx,
                                                       CS_IDX_TX_VALUE_VAL,
                                                       &temp[PACKET_SIZE],
                                                       length - PACKET_SIZE);
                    }
                    else
                    {
                        CustomService_SendNotification(ble_env.conidx,
                                                       CS_IDX_TX_VALUE_VAL,
                                                       temp,
                                                       length);
                    }
                }
            }
        }

        /* Refresh the watchdog timer */
        Sys_Watchdog_Refresh();

        /* Wait for an event before executing the scheduler again */
        SYS_WAIT_FOR_EVENT;
    }
}

```

初始化

```
app.c

App_Initialize();
```

### 串口初始化

```
void UART_Initialize(void)
{
    static uint32_t UARTTXBuffer[BUFFER_SIZE * 2];

    /* RX buffer with accounted overhead size */
    static uint32_t UARTRXBuffer[BUFFER_SIZE];

    /* Setup a UART interface on DIO5 (TX) and DIO4 (RX), routing data using
     * DMA channels 0 (TX), 1 (RX) to two defined buffers - don't setup DMA
     * channel 0 since there isn't any data to transmit yet. */
    Sys_DMA_ChannelDisable(DMA_TX_NUM);
    Sys_DMA_ChannelDisable(DMA_RX_NUM);

    gUARTTXData = UARTTXBuffer;
    gNextData = 0;
	// 配置成DMA，如果要更改串口的接受和发送引脚，直接鞥改CFG_DIO_TX_UART和CFG_DIO_RXD_UART的定义即可
    /* Configure TX and RX pin with correct current values */
    Sys_UART_DIOConfig(DIO_6X_DRIVE | DIO_WEAK_PULL_UP | DIO_LPF_ENABLE,
                       CFG_DIO_TXD_UART, CFG_DIO_RXD_UART);
	// 使能时钟和波特率
    /* Enable device UART port */
    Sys_UART_Enable(CFG_SYS_CLK, CFG_UART_BAUD_RATE, UART_DMA_MODE_ENABLE);

    /* Configure the RX DMA channel, clear status and enable DMA channel */
    Sys_DMA_ClearChannelStatus(DMA_RX_NUM);
    Sys_DMA_ChannelConfig(DMA_RX_NUM, DMA_RX_CFG, BUFFER_SIZE,
                          (BUFFER_SIZE / 2), (uint32_t)&UART->RX_DATA,
                          (uint32_t)UARTRXBuffer);
    NVIC_EnableIRQ(DMA1_IRQn);
}

```

### BLE初始化

```
void BLE_Initialize(void)
{
    struct gapm_reset_cmd *cmd;
    // 默认mac地址
    uint8_t default_addr[BDADDR_LENGTH] = PRIVATE_BDADDR;

    /* Seed the random number generator */
    srand(1);

    /* Set radio clock accuracy in ppm */
    BLE_DeviceParam_Set_ClockAccuracy(RADIO_CLOCK_ACCURACY);

    /* Initialize the kernel and Bluetooth stack */
    Kernel_Init(0);
    BLE_InitNoTL(0);
    BLE_Reset();

    /* Enable the Bluetooth related interrupts needed */
    NVIC_EnableIRQ(BLE_EVENT_IRQn);
    NVIC_EnableIRQ(BLE_RX_IRQn);
    NVIC_EnableIRQ(BLE_CRYPT_IRQn);
    NVIC_EnableIRQ(BLE_ERROR_IRQn);
    NVIC_EnableIRQ(BLE_SW_IRQn);
    NVIC_EnableIRQ(BLE_GROSSTGTIM_IRQn);
    NVIC_EnableIRQ(BLE_FINETGTIM_IRQn);
    NVIC_EnableIRQ(BLE_CSCNT_IRQn);
    NVIC_EnableIRQ(BLE_SLP_IRQn);

    /* Reset the Bluetooth environment */
    memset(&ble_env, 0, sizeof(ble_env));

    /* Initialize task state */
    ble_env.state = APPM_INIT;

    /* Set Bluetooth device type and address: depending on the device address
     * type selected by the application, either a public or private address is
     * used:
     *     - In case public address type is selected: (i) If a public address is
     *       provided by the application, it is used; (ii) Otherwise the device
     *       public address available in DEVICE_INFO_BLUETOOTH_ADDR (located in
     *       NVR3) is used. If no public address is available in NVR3, a default
     *       public address pre-defined in the stack is used instead.
     *     - In case private address type is selected: the private address
     *       provided by the application is used. */
#if (BD_ADDRESS_TYPE == BD_TYPE_PUBLIC)
    bdaddr_type = GAPM_CFG_ADDR_PUBLIC;
    if (Device_Param_Read(PARAM_ID_PUBLIC_BLE_ADDRESS, (uint8_t *)&bdaddr))
    {
        memcpy(bdaddr, default_addr, sizeof(uint8_t) * BDADDR_LENGTH);
    }
    else
    {
        memcpy(bdaddr, &co_default_bdaddr, sizeof(uint8_t) * BDADDR_LENGTH);
    }
#else    /* if (BD_ADDRESS_TYPE == BD_TYPE_PUBLIC) */
    memcpy(bdaddr, default_addr, sizeof(uint8_t) * BDADDR_LENGTH);
    bdaddr_type = GAPM_CFG_ADDR_PRIVATE;
#endif    /* if (BD_ADDRESS_TYPE == BD_TYPE_PUBLIC) */

    /* Initialize GAPM configuration command to initialize the stack */
    gapmConfigCmd =
        malloc(sizeof(struct gapm_set_dev_config_cmd));
    gapmConfigCmd->operation      = GAPM_SET_DEV_CONFIG;
    gapmConfigCmd->role = GAP_ROLE_PERIPHERAL;		// 作为蓝牙的外围设备工作，也就是client
    memcpy(gapmConfigCmd->addr.addr, bdaddr, BDADDR_LENGTH);
    gapmConfigCmd->addr_type = bdaddr_type;
    gapmConfigCmd->renew_dur = RENEW_DUR;
    memset(&gapmConfigCmd->irk.key[0], 0, KEY_LEN);
    gapmConfigCmd->pairing_mode = GAPM_PAIRING_DISABLE;
    gapmConfigCmd->gap_start_hdl = 0;
    gapmConfigCmd->gatt_start_hdl = 0;
    gapmConfigCmd->max_mtu = MTU_MAX;
    gapmConfigCmd->max_mps = MPS_MAX;
    gapmConfigCmd->att_cfg = ATT_CFG;
    gapmConfigCmd->sugg_max_tx_octets = TX_OCT_MAX;
    gapmConfigCmd->sugg_max_tx_time = TX_TIME_MAX;
    gapmConfigCmd->tx_pref_rates = GAP_RATE_ANY;
    gapmConfigCmd->rx_pref_rates = GAP_RATE_ANY;
    gapmConfigCmd->max_nb_lecb = 0x0;
    gapmConfigCmd->audio_cfg = 0;

    /* Reset the stack */
    cmd = KE_MSG_ALLOC(GAPM_RESET_CMD, TASK_GAPM, TASK_APP, gapm_reset_cmd);
    cmd->operation = GAPM_RESET;

    /* Send the message */
    ke_msg_send(cmd);
}

```

### 线程初始化

```
void App_Env_Initialize(void)
{
    /* Reset the application manager environment */
    memset(&app_env, 0, sizeof(app_env));
    // 创建task
    /* Create the application task handler */
    ke_task_create(TASK_APP, &TASK_DESC_APP);

    /* Initialize the custom service environment */
    CustomService_Env_Initialize();

    /* Initialize the battery service server environment */
    Bass_Env_Initialize();
}

const struct ke_task_desc TASK_DESC_APP = {
    NULL, &appm_default_handler,
    appm_state, APPM_STATE_MAX,
    APP_IDX_MAX
};

/* State and event handler definition */
const struct ke_msg_handler appm_default_state[] =
{
    /* Note: Put the default handler on top as this is used for handling any
     *       messages without a defined handler */
    { KE_MSG_DEFAULT_HANDLER, (ke_msg_func_t)Msg_Handler },
    BLE_MESSAGE_HANDLER_LIST,		// 蓝牙gap,gatt属性设置，各种连接事件设置
    BASS_MESSAGE_HANDLER_LIST,		// 电池相关服务
    CS_MESSAGE_HANDLER_LIST,		// 读写操作的回调函数
    APP_MESSAGE_HANDLER_LIST,		// 用户app的task
    UART_MESSAGE_HANDLER_LIST		// 串口相关
};

/* Use the state and event handler definition for all states. */
const struct ke_state_handler appm_default_handler
    = KE_STATE_HANDLER(appm_default_state);

```

* 蓝牙gap,gatt属性设置

```
#define BLE_MESSAGE_HANDLER_LIST                                              \
    DEFINE_MESSAGE_HANDLER(GAPM_CMP_EVT, GAPM_CmpEvt),                        \
    DEFINE_MESSAGE_HANDLER(GAPM_PROFILE_ADDED_IND, GAPM_ProfileAddedInd),     \
    DEFINE_MESSAGE_HANDLER(GAPC_CONNECTION_REQ_IND, GAPC_ConnectionReqInd),   \
    DEFINE_MESSAGE_HANDLER(GAPC_CMP_EVT, GAPC_CmpEvt),                        \
    DEFINE_MESSAGE_HANDLER(GAPC_DISCONNECT_IND, GAPC_DisconnectInd),          \
    DEFINE_MESSAGE_HANDLER(GAPC_GET_DEV_INFO_REQ_IND, GAPC_GetDevInfoReqInd), \
    DEFINE_MESSAGE_HANDLER(GAPC_PARAM_UPDATED_IND, GAPC_ParamUpdatedInd),     \
    DEFINE_MESSAGE_HANDLER(GAPC_PARAM_UPDATE_REQ_IND, GAPC_ParamUpdateReqInd) \

```

开始广播 ,设置gap, gatt的参数

```
Advertising_Start();

void Advertising_Start(void)
{
    uint8_t device_name_length;
    uint8_t device_name_avail_space;
    uint8_t scan_rsp[SCAN_RSP_DATA_LEN] = APP_SCNRSP_DATA;
    uint8_t company_id[APP_COMPANY_ID_DATA_LEN] = APP_COMPANY_ID_DATA;

    /* Prepare the GAPM_START_ADVERTISE_CMD message */
    struct gapm_start_advertise_cmd *cmd;

    /* If the application is ready, start advertising */
    if (ble_env.state == APPM_READY)
    {
        /* Prepare the start advertisment command message */
        cmd = KE_MSG_ALLOC(GAPM_START_ADVERTISE_CMD, TASK_GAPM, TASK_APP,
                           gapm_start_advertise_cmd);
        cmd->op.addr_src = GAPM_STATIC_ADDR;
        cmd->channel_map = APP_ADV_CHMAP;	// 广播信道

        cmd->intv_min = APP_ADV_INT_MIN;	// 最大连接时间间隔
        cmd->intv_max = APP_ADV_INT_MAX;	// 最小连接时间间隔

        cmd->op.code = GAPM_ADV_UNDIRECT;	// 不定向的广播
        cmd->op.state = 0;
        cmd->info.host.mode = GAP_GEN_DISCOVERABLE;		// 可发现的广播
        cmd->info.host.adv_filt_policy = 0;

        /* Set the scan response data */
        cmd->info.host.scan_rsp_data_len = APP_SCNRSP_DATA_LEN;	// 数据长度
        memcpy(&cmd->info.host.scan_rsp_data[0],
               scan_rsp, cmd->info.host.scan_rsp_data_len);

        /* Get remaining space in the advertising data -
         * 2 bytes are used for name length/flag */
        cmd->info.host.adv_data_len = 0;
        device_name_avail_space = (ADV_DATA_LEN - 3) - 2;
        // 查看广播数据是否有空间
        /* Check if data can be added to the advertising data */
        if (device_name_avail_space > 0)
        {
        	// 添加设备名称到广播数据中
            /* Add as much of the device name as possible */
            device_name_length = strlen(APP_DFLT_DEVICE_NAME);	// 广播名称“Peripheral_Server_UART”
            if (device_name_length > 0)
            {
                /* Check available space */
                device_name_length = co_min(device_name_length,
                                            device_name_avail_space);
                cmd->info.host.adv_data[cmd->info.host.adv_data_len] =
                    device_name_length + 1;

                /* Fill device name flag */
                cmd->info.host.adv_data[cmd->info.host.adv_data_len + 1] =
                    APP_DEVICE_NAME_FLAG;
                // 将名称保存
                /* Copy device name */
                memcpy(&cmd->info.host.adv_data[cmd->info.host.adv_data_len +
                                                2],
                       APP_DFLT_DEVICE_NAME, device_name_length);

                /* Update advertising data length */
                cmd->info.host.adv_data_len += (device_name_length + 2);
            }

            /* If there is still space, add the company ID */
            if (((ADV_DATA_LEN - 3) - cmd->info.host.adv_data_len - 2) >=
                APP_COMPANY_ID_DATA_LEN)
            {
                memcpy(&cmd->info.host.adv_data[cmd->info.host.adv_data_len],
                       company_id, APP_COMPANY_ID_DATA_LEN);
                cmd->info.host.adv_data_len += APP_COMPANY_ID_DATA_LEN;
            }
        }

        /* Send the message */
        ke_msg_send(cmd);

        /* Set the state of the task to APPM_ADVERTISING  */
        ble_env.state = APPM_ADVERTISING;
    }
}

```

用户添加自定义的服务

```
ble_std.c

bool Service_Add(void)
{
    /* Check if another should be added in the database */
    if (appm_add_svc_func_list[ble_env.next_svc] != NULL)
    {
        /* Call the function used to add the required service */
        appm_add_svc_func_list[ble_env.next_svc] ();

        /* Select the next service to add */
        ble_env.next_svc++;
        return (true);
    }

    return (false);
}


/* List of functions used to create the database */
const appm_add_svc_func_t appm_add_svc_func_list[] = {
    SERVICE_ADD_FUNCTION_LIST, NULL
};

app.h

#define SERVICE_ADD_FUNCTION_LIST                        \
    DEFINE_SERVICE_ADD_FUNCTION(Batt_ServiceAdd_Server), \
    DEFINE_SERVICE_ADD_FUNCTION(CustomService_ServiceAdd)


ble_custom.c

void CustomService_ServiceAdd(void)
{
    struct gattm_add_svc_req *req = KE_MSG_ALLOC_DYN(GATTM_ADD_SVC_REQ,
                                                     TASK_GATTM, TASK_APP,
                                                     gattm_add_svc_req,
                                                     CS_IDX_NB * sizeof(struct
                                                                        gattm_att_desc));

    uint8_t i;
    // 用户自己定义的service的UUID
    const uint8_t svc_uuid[ATT_UUID_128_LEN] = CS_SVC_UUID;

    const struct gattm_att_desc att[CS_IDX_NB] =
    {
        /* Attribute Index  = Attribute properties: UUID,
         *                                          Permissions,
         *                                          Max size,
         *                                          Extra permissions */

        /* TX Characteristic */
        /* TX Characteristic */
        [CS_IDX_TX_VALUE_CHAR]     = ATT_DECL_CHAR(),
		// 用户自己定义service里面的属性的UUID， 权限，可读，并且是notify
        [CS_IDX_TX_VALUE_VAL]      = ATT_DECL_CHAR_UUID_128(CS_CHARACTERISTIC_TX_UUID,
                                                            PERM(RD, ENABLE) | PERM(NTF, ENABLE),
                                                            CS_TX_VALUE_MAX_LENGTH),
		// CCC: client characteristic configuration, 可读，读出来的值是“Notifications enabled"
        [CS_IDX_TX_VALUE_CCC]      = ATT_DECL_CHAR_CCC(),
		// characteristic User Description, 可读, 读取的值是"TX_VALUE"
        [CS_IDX_TX_VALUE_USR_DSCP] = ATT_DECL_CHAR_USER_DESC(CS_USER_DESCRIPTION_MAX_LENGTH),

        /* RX Characteristic */
        [CS_IDX_RX_VALUE_CHAR]     = ATT_DECL_CHAR(),
		// 用户定义的service另外一个属性的UUID，具有读权限，写权限
        [CS_IDX_RX_VALUE_VAL]      = ATT_DECL_CHAR_UUID_128(CS_CHARACTERISTIC_RX_UUID,
                                                            PERM(RD, ENABLE)
                                                            | PERM(WRITE_REQ, ENABLE) | PERM(WRITE_COMMAND, ENABLE),
                                                            CS_RX_VALUE_MAX_LENGTH),
		// 可读，值是"Notifications or indictions disabled"
        [CS_IDX_RX_VALUE_CCC]      = ATT_DECL_CHAR_CCC(),
		// 可读，"RX_VALUE"
        [CS_IDX_RX_VALUE_USR_DSCP] = ATT_DECL_CHAR_USER_DESC(CS_USER_DESCRIPTION_MAX_LENGTH),
    };

    /* Fill the add custom service message */
    req->svc_desc.start_hdl = 0;
    req->svc_desc.task_id = TASK_APP;
    req->svc_desc.perm = PERM(SVC_UUID_LEN, UUID_128);
    req->svc_desc.nb_att = CS_IDX_NB;

    memcpy(&req->svc_desc.uuid[0], &svc_uuid[0], ATT_UUID_128_LEN);

    for (i = 0; i < CS_IDX_NB; i++)
    {
        memcpy(&req->svc_desc.atts[i], &att[i],
               sizeof(struct gattm_att_desc));
    }

    /* Send the message */
    ke_msg_send(req);
}
```

读写操作回调函数

写操作，会将写过来的数据通过串口发送出去。

```
#define CS_MESSAGE_HANDLER_LIST                                     \
    DEFINE_MESSAGE_HANDLER(GATTC_READ_REQ_IND, GATTC_ReadReqInd),   \
    DEFINE_MESSAGE_HANDLER(GATTC_WRITE_REQ_IND, GATTC_WriteReqInd), \
    DEFINE_MESSAGE_HANDLER(GATTM_ADD_SVC_RSP, GATTM_AddSvcRsp),     \
    DEFINE_MESSAGE_HANDLER(GATTC_CMP_EVT, GATTC_CmpEvt)

ble_custom.c

int GATTC_WriteReqInd(ke_msg_id_t const msg_id,
                      struct gattc_write_req_ind const *param,
                      ke_task_id_t const dest_id,
                      ke_task_id_t const src_id)
{
    struct gattc_write_cfm *cfm = KE_MSG_ALLOC(GATTC_WRITE_CFM,
                                               KE_BUILD_ID(TASK_GATTC,
                                                           ble_env.conidx),
                                               TASK_APP, gattc_write_cfm);

    uint8_t status = GAP_ERR_NO_ERROR;
    uint16_t attnum = 0;
    uint8_t *valptr = NULL;
    overflow_packet_t *traverse = NULL;
    int flag;

    /* Check that offset is not zero */
    if (param->offset)
    {
        status = ATT_ERR_INVALID_OFFSET;
    }

    /* Set the attribute handle using the attribute index
     * in the custom service */
    if (param->handle > cs_env.start_hdl)
    {
        attnum = (param->handle - cs_env.start_hdl - 1);
    }
    else
    {
        status = ATT_ERR_INVALID_HANDLE;
    }

    /* If there is no error, save the requested attribute value */
    if (status == GAP_ERR_NO_ERROR)
    {
        switch (attnum)
        {
            case CS_IDX_RX_VALUE_VAL:
            {
            	Sys_GPIO_Toggle(7);
                valptr = (uint8_t *)&cs_env.rx_value;
                cs_env.rx_value_changed = 1;
            }
            break;

            case CS_IDX_RX_VALUE_CCC:
            {

                valptr = (uint8_t *)&cs_env.rx_cccd_value;
            }
            break;

            case CS_IDX_TX_VALUE_CCC:
            {
                valptr = (uint8_t *)&cs_env.tx_cccd_value;
            }
            break;
            default:
            {
                status = ATT_ERR_WRITE_NOT_PERMITTED;
            }
            break;
        }
    }

    if (valptr != NULL)
    {
        memcpy(valptr, param->value, param->length);
    }

    cfm->handle = param->handle;
    cfm->status = status;

    /* Send the message */
    ke_msg_send(cfm);

    if (attnum == CS_IDX_RX_VALUE_VAL)
    {
        flag = 0;

        /* Start by trying to queue up any previously unhandled packets. If we
         * can't queue them all, set a flag to indicate that we need to queue
         * the new packet too. */
        // 如果有没有处理的数据
        while ((unhandled_packets != NULL) && (flag == 0))
        {
        	// 串口发送数据
            if (UART_FillTXBuffer(unhandled_packets->length,
                                  unhandled_packets->data) !=
                UART_ERRNO_OVERFLOW)
            {
                /* Remove a successfully queued packet from the list of unqueued
                 * packets. */
            	// 处理完成，从链表中删除
                unhandled_packets = removeNode(unhandled_packets);
            }
            else
            {
            	// 处理异常，还有数据没处理，把标志位置1
                flag = 1;
            }
        }

        /* If we don't have any (more) outstanding packets, attempt to queue the
         * current packet. If this packet is successfully queued, exit. */
        if (flag == 0)
        {
        	// 发送当前收到的数据
            if (UART_FillTXBuffer(param->length, valptr) != UART_ERRNO_OVERFLOW)
            {
            	// 数据发送成功
                return (KE_MSG_CONSUMED);
            }
        }
        /* We couldn't empty the list or we couldn't queue the new packet. In
         * both cases we need to save the current packet at the end of the list,
         * then exit. */
        // 数据没法成功，链表不为空
        if (unhandled_packets != NULL)
        {
        	// 数据插入到链表最后
            traverse = unhandled_packets;
            while (traverse->next != NULL)
            {
                traverse = traverse->next;
            }
            traverse->next = createNode(param->length, valptr);
        }
        else
        {
        	//链表为空，数据没发送完，将数据放在链表头
            unhandled_packets = createNode(param->length, valptr);
        }
    }

    return (KE_MSG_CONSUMED);
}
```

* 定时器任务

```
#define APP_MESSAGE_HANDLER_LIST \
    DEFINE_MESSAGE_HANDLER(APP_TEST_TIMER, APP_Timer), \
    DEFINE_MESSAGE_HANDLER(LED_TIMER, LED_Timer)

int APP_Timer(ke_msg_id_t const msg_id,
              void const *param,
              ke_task_id_t const dest_id,
              ke_task_id_t const src_id)
{
    uint16_t level;
    // 200MS进入这个函数一次
    /* Restart timer */
    ke_timer_set(APP_TEST_TIMER, TASK_APP, TIMER_200MS_SETTING);

    /* Every six seconds report that the custom service TX value changed
     * (notification simulation) */
    cs_env.cnt_notifc++;
    if (cs_env.cnt_notifc == 30)
    {
    	// 每6s中报告一次TX值改变了
        cs_env.cnt_notifc = 0;
        cs_env.tx_value_changed = 1;
    }
    // 计算ADC的值（电池的电压值），并用百分比进行表示。
    /* Calculate the battery level as a percentage, scaling the battery
     * voltage between 1.4V (max) and 1.1V (min) */
    level = ((ADC->DATA_TRIM_CH[0] - VBAT_1p1V_MEASURED) * 100
             / (VBAT_1p4V_MEASURED - VBAT_1p1V_MEASURED));
    level = ((level >= 100) ? 100 : level);

    /* Add to the current sum and increment the number of reads,
     * calculating the average over 16 voltage reads */
    // 计算16电池电量总和
    app_env.sum_batt_lvl += level;
    app_env.num_batt_read++;
    if (app_env.num_batt_read == 16)
    {
    	// 除以16，算出平均值
        if ((app_env.sum_batt_lvl >> 4) != app_env.batt_lvl)
        {
        	// 标志位置1，可以发送电池的电量
            app_env.send_batt_ntf = 1;
        }

        if (ble_env.state == APPM_CONNECTED && bass_support_env.enable)
        {
        	// 保存电池电量
            app_env.batt_lvl = (app_env.sum_batt_lvl >> 4);
        }

        app_env.num_batt_read = 0;
        app_env.sum_batt_lvl = 0;
    }

    return (KE_MSG_CONSUMED);
}

int LED_Timer(ke_msg_id_t const msg_id,
              void const *param,
              ke_task_id_t const dest_id,
              ke_task_id_t const src_id)
{
	// 连接状态，LED灯一直亮
    if (ble_env.state == APPM_CONNECTED)
    {
        Sys_GPIO_Set_High(LED_DIO_NUM);
    }
    // 广播状态，LED灯闪烁
    else if (ble_env.state == APPM_ADVERTISING)
    {
        Sys_GPIO_Toggle(LED_DIO_NUM);
    }
    // 其他状态，LED灯灭
    else
    {
        Sys_GPIO_Set_Low(LED_DIO_NUM);
    }
    // 设置定时器，ID:LED_TIMER， TASK:TASK_APP, delay: 200MS
    // 200MS之后，又来处理ID是LED_TIMER的handler，处理函数就是LED_Timer(),再次进入这个函数
    /* Reschedule timer */
    ke_timer_set(LED_TIMER, TASK_APP, TIMER_200MS_SETTING);

    return (KE_MSG_CONSUMED);
}
```
* UART 设置

```

#define UART_MESSAGE_HANDLER_LIST \
    DEFINE_MESSAGE_HANDLER(UART_TEST_TIMER, UART_Timer)

uart.c


int UART_Timer(ke_msg_id_t const msg_id,
               void const *param,
               ke_task_id_t const dest_id,
               ke_task_id_t const src_id)
{
    /* Restart timer */
    ke_timer_set(UART_TEST_TIMER, TASK_APP, TIMER_20MS_SETTING);
    timerFlag = 1;

    return (KE_MSG_CONSUMED);
}

```

串口接收函数

```
uint32_t UART_EmptyRXBuffer(uint8_t *data)
{
    uint32_t size;
    uint32_t temp;
    unsigned int i;

    /* Check if the start interrupt event occured and then setup the RX next
     * data pointer */
    // 这里应该是开机第一次的时候来判断，开机的时候gNextData=0
    if (((Sys_DMA_Get_ChannelStatus(DMA_RX_NUM) & DMA_START_INT_STATUS) != 0) &&
        (gNextData == 0))
    {
        gNextData = DMA->DEST_BASE_ADDR[DMA_RX_NUM];
    }
    // timeFlag在串口的定时器中设置为1，每20ms设置一次。
    else if (timerFlag == 0)
    {
    	//timeFlag=0时，每次都更新temp的值，因为DMA的数据一直在增加，所以temp的值也在增加
    	// timeFlag=1时，不更新temp的值，把数据取出来到data中
        temp = DMA->NEXT_DEST_ADDR[DMA_RX_NUM];
        if (gNextData < temp)
        {
            size = (temp - gNextData) >> 2;
        }
        else if (gNextData > temp)
        {
            size = (temp + ((BUFFER_SIZE << 2) - gNextData)) >> 2;
        }
        else
        {
            return (0);
        }

        /* Keep polling if size is smaller than maximum possible size -
         * (BUFFER_SIZE - 3*POLL_SIZE). POLL_SIZE represents bytes that can pile
         * up in between polls and '3*' is a safety factor in case the baud rate
         * isn't correct. */
        // 查看size是否查过了最大值，没超过就不更新
        if (size < (BUFFER_SIZE - (3 * POLL_SIZE)))
        {
            return (0);
        }
    }

    /* Set temp so that processing in this function is atomic */
    temp = DMA->NEXT_DEST_ADDR[DMA_RX_NUM];

    /* If gNextData is not initialized or matches the next address location
     * to be written return a size of 0. */
    // 接受长度为0，直接返回
    if (gNextData == temp)
    {
        return (0);
    }

    /* Otherwise convert the size from the input word size (32 bits) to the
     * expected word size (8 bits) for UART transfers, and copy the received
     * words from the DMA buffer. Keep gNextData up to date as we read data from
     * the buffer so its set for the next time we enter this function */
    // 取出数据
    if (gNextData < temp)
    {
        /* Handle copying out data that does not wrap around the end of the
         * buffer where the DMA is copying data to. */
        // 获得数据大小，因为是是4字节对齐，所以要除4.
    	size = (temp - gNextData) >> 2;
    	// 从DMA中取出数据，保存到data中
        for (i = 0; i < size; i++)
        {
            data[i] = *(uint8_t *)gNextData;
            gNextData = gNextData + 4;
        }
    }
    else
    {
    	// 如果数据回环了，也就是数据超过了DMA地址的最大值，就又从最小的地址开始存数据
        /* Handle copying out data that wraps around the end of the
         * buffer where the DMA is copying data to. */
    	// 计算大小
        size = (temp + ((BUFFER_SIZE << 2) - gNextData)) >> 2;
        temp = ((BUFFER_SIZE << 2) - (gNextData -
                                      DMA->DEST_BASE_ADDR[DMA_RX_NUM])) >> 2;
        // 当前地址到DMA结尾的
        for (i = 0; i < temp; i++)
        {
            data[i] = *(uint8_t *)gNextData;
            gNextData = gNextData + 4;
        }
        // 开头一部分的数据
        gNextData = DMA->DEST_BASE_ADDR[DMA_RX_NUM];
        if (size - temp > 0)
        {
            for (i = 0; i < (size - temp); i++)
            {
                data[temp + i] = *(uint8_t *)gNextData;
                gNextData = gNextData + 4;
            }
        }
    }

    timerFlag = 0;
    return (size);
}
```
