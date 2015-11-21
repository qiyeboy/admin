/*
 * Copyright (c) 2009-2012 Xilinx, Inc.  All rights reserved.
 *
 * Xilinx, Inc.
 * XILINX IS PROVIDING THIS DESIGN, CODE, OR INFORMATION "AS IS" AS A
 * COURTESY TO YOU.  BY PROVIDING THIS DESIGN, CODE, OR INFORMATION AS
 * ONE POSSIBLE   IMPLEMENTATION OF THIS FEATURE, APPLICATION OR
 * STANDARD, XILINX IS MAKING NO REPRESENTATION THAT THIS IMPLEMENTATION
 * IS FREE FROM ANY CLAIMS OF INFRINGEMENT, AND YOU ARE RESPONSIBLE
 * FOR OBTAINING ANY RIGHTS YOU MAY REQUIRE FOR YOUR IMPLEMENTATION.
 * XILINX EXPRESSLY DISCLAIMS ANY WARRANTY WHATSOEVER WITH RESPECT TO
 * THE ADEQUACY OF THE IMPLEMENTATION, INCLUDING BUT NOT LIMITED TO
 * ANY WARRANTIES OR REPRESENTATIONS THAT THIS IMPLEMENTATION IS FREE
 * FROM CLAIMS OF INFRINGEMENT, IMPLIED WARRANTIES OF MERCHANTABILITY
 * AND FITNESS FOR A PARTICULAR PURPOSE.
 *
 */

/*
 * helloworld.c: simple test application
 *
 * This application configures UART 16550 to baud rate 9600.
 * PS7 UART (Zynq) is not initialized by this application, since
 * bootrom/bsp configures it to baud rate 115200
 *
 * ------------------------------------------------
 * | UART TYPE   BAUD RATE                        |
 * ------------------------------------------------
 *   uartns550   9600
 *   uartlite    Configurable only in HW design
 *   ps7_uart    115200 (configured by bootrom/bsp)
 */

#include <stdio.h>
#include "platform.h"
#include "axi_tran.h"
#include "xil_io.h"
#include "crossbar_define.h"
#include "crossbar_1588.h"
#include "crossbar_interrupt.h"

Sync m_sync;
Follow_Up m_follow_up;
Delay_Req delay_req;
Delay_Resp delay_resp;
u32 interruptflag = 0;
Time_1588 sec_time_1588[4];
int state = -1;//全局状态 0Sync,1Follo_UP 2,delay_req 3 delay_resp
void delay_s(u32 sec){
	u32 i,j;
	for(j=0;j<sec;j++){
		for(i=0;i<100000000;i++);
	}
}


const unsigned int frame_data_ptp[]={
    0x55555555,0x5555555D,0x0062E97C,0x57652143,
    0x65878967,0x88f70002,0x00000000,0x00000000,
    0x00000000,0x00000000,0x00000000,0x00000000,
    0x00000000,0x0000007f,0x00010000,0x00010000,
    0x0001ffff,0xffff0000,0xffffffff
};
const unsigned int frame_data_zmac[]={
    0x0fffffff,0x1fffffff,0x2fffffff,0x3fffffff,
    0x4fffffff,0x5fffffff,0x6fffffff,0x7fffffff,
    0x8fffffff,0x9fffffff,0xafffffff,0xbfffffff
};


unsigned int insert_end,frame_pri,frame_len,des_port_id,insert_reserved;
unsigned int insert_ctrl_data;
/**
 * 系统初始化内容
 */
void init(){

	init_platform();
	init1588Config();//初始化1588配置
	init_1588_Interrupt();//初始化1588中断


}
int main()
{
	init();
	initSync(&m_sync);
	initFollow_Up(&m_follow_up);
	initDelay_Resp(&delay_resp);
    initDelay_Req(&delay_req);
	InsertCtlInfor ctlInfor;

#if 0//主时钟，之后可以用拨码开关

    while(1){//主时钟程序
    	//发送Sync帧

    	ctlInfor.insert_end=0;
    	ctlInfor.frame_pri = 0;
    	ctlInfor.frame_len = sizeof(Sync);
    	ctlInfor.des_port_id = 4;
    	ctlInfor.insert_reserved = 0;
    	state = 0;
    	sendSync(&ctlInfor,&m_sync);
    	while(state == 0);//等待状态变化，当state变成1时候开始发送Follow_Up
    	//读取t1(发送时间),将时间信息打包到FolloUp中
    	u32 timeArray[4];
    	Time_1588 time_1588;
    	tx_get_128(timeArray);
    	time_1588 = transto1588(timeArray);
    	pack1588_to_FollowUp(&time_1588,&m_follow_up);
    	//发送FolloUp
    	ctlInfor.insert_end=0;
    	ctlInfor.frame_pri = 0;
    	ctlInfor.frame_len = sizeof(Follow_Up);
    	ctlInfor.des_port_id = 4;
    	ctlInfor.insert_reserved = 0;
    	sendFollow_Up(&ctlInfor,&m_follow_up);
    	while(state == 1 || state == 2);//等待发送delayResp
    	//发送delayResp
    	ctlInfor.insert_end=0;
    	ctlInfor.frame_pri = 0;
    	ctlInfor.frame_len = sizeof(Delay_Resp);
    	ctlInfor.des_port_id = 4;
    	ctlInfor.insert_reserved = 0;
    	sendDelay_Resp(&ctlInfor,&delay_resp);

    	delay_s(2);//延时两秒


    }
#else

    while(1){

    	if(state == 1){//接下来发送delay_req
        	ctlInfor.insert_end=0;
        	ctlInfor.frame_pri = 0;
        	ctlInfor.frame_len = sizeof(Delay_Req);
        	ctlInfor.des_port_id = 4;
        	ctlInfor.insert_reserved = 0;
        	sendDelay_Req(&ctlInfor,&delay_req);
    	}



    }

#endif




    /*AXI_TRAN_mWriteReg(BaseAddress, CPU_Addr_Wr, INSERT_REQ);
    insert_flag = AXI_TRAN_mReadReg(BaseAddress, CPU_Data_Rd);
    if(insert_flag){
        xil_printf("insert_frame\n\r");
        insert_end  = 0;
        frame_pri = 0;
        frame_len = 73;
        des_port_id = 4;
        insert_reserved = 0;
        insert_ctrl_data = (insert_end<<31)|(frame_pri<<29)|(frame_len<<18)|(des_port_id<<10)|insert_reserved;
        AXI_TRAN_mWriteReg(BaseAddress, CPU_Addr_Wr, INSERT_CTRL);
        AXI_TRAN_mWriteReg(BaseAddress, CPU_Data_Wr, insert_ctrl_data);
        for(insert_i=1;insert_i<=(sizeof(frame_data)/sizeof(unsigned int));insert_i++){
            AXI_TRAN_mWriteReg(BaseAddress, CPU_Addr_Wr, (INSERT_DATA+insert_i));
            AXI_TRAN_mWriteReg(BaseAddress, CPU_Data_Wr, frame_data[insert_i-1]);
        }
        insert_end  = 1;
        insert_ctrl_data = (insert_end<<31)|(frame_pri<<29)|(frame_len<<18)|(des_port_id<<10)|insert_reserved;
        AXI_TRAN_mWriteReg(BaseAddress, CPU_Addr_Wr, INSERT_CTRL);
        AXI_TRAN_mWriteReg(BaseAddress, CPU_Data_Wr, insert_ctrl_data);
        insert_end  = 0;
        insert_ctrl_data = (insert_end<<31)|(frame_pri<<29)|(frame_len<<18)|(des_port_id<<10)|insert_reserved;
        AXI_TRAN_mWriteReg(BaseAddress, CPU_Addr_Wr, INSERT_CTRL);
        AXI_TRAN_mWriteReg(BaseAddress, CPU_Data_Wr, insert_ctrl_data);

    }

    AXI_TRAN_mWriteReg(BaseAddress, CPU_Addr_Wr, 0xffffffff);*/

//    if(time_32bit_s != time_32bit_s_tmp){
//        xil_printf("16bit s : %d \n\r",time_16bit_s);
//        xil_printf("32bit s : %d \n\r",time_32bit_s);
//        xil_printf("32bit ns : %d \n\r",time_32bit_ns);
//        xil_printf("8bit nsf : %d \n\r",time_8bit_nsf);
//        xil_printf("8bit stat : %x \n\r",rx_quene_state);
//        xil_printf("first 32bit timestamp : %d \n\r",first_32bit_timestamp);
//        xil_printf("second 32bit timestamp : %d \n\r",second_32bit_timestamp);
//        xil_printf("third 32bit timestamp : %d \n\r",third_32bit_timestamp);
//        xil_printf("fourth 32bit timestamp : %x \n\r",fourth_32bit_timestamp);
//    }
//
//    time_32bit_s_tmp = time_32bit_s;
//    }

    return 0;
}
/*
 * crossbar_interrupt.c
 *
 *  Created on: 2015-11-13
 *      Author: Xaxdus
 */

#include"xil_io.h"
#include"xstatus.h"
#include "crossbar_interrupt.h"
#include "crossbar_1588.h"

static INTC Intc;//全局的INTC
extern Delay_Resp delay_resp;
extern u32 interruptflag;
extern  int state;
extern Time_1588 sec_time_1588[4];
/**
 * 	初始化中断配置
 */
int init_1588_Interrupt(){
	int result;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Config* IntcConfig;
	/* Initializethe interrupt controller driver so that it is ready to
	* use.
	* */
	IntcConfig =XScuGic_LookupConfig(INTC_DEVICE_ID);
	if (IntcConfig== NULL)
	{
		return XST_FAILURE;
	}
	/* Initializethe SCU and GIC to enable the desired interrupt
	* configuration.*/
	result =XScuGic_CfgInitialize(IntcInstancePtr, IntcConfig,
	IntcConfig->CpuBaseAddress);
	if (result !=XST_SUCCESS)
	{
		return XST_FAILURE;
	}
	XScuGic_SetPriorityTriggerType(IntcInstancePtr,crossbar_1588_interruptID,
	0xA0, 0x3);
	/* Connect the interrupt handler that will be called when an
	* interrupt occurs for the device. */
	result =XScuGic_Connect(IntcInstancePtr, crossbar_1588_interruptID,
	(Xil_ExceptionHandler)deal_1588_interrupt, 0);
	if (result !=XST_SUCCESS)
	{
		return result;
	}
	/* Enable the interrupt for the 1588 controller device. */
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);
	/* Initializethe exception table and register the interrupt controller
	* handler withthe exception table. */
	Xil_ExceptionInit();
	Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,
	(Xil_ExceptionHandler)INTC_HANDLER,IntcInstancePtr);
	/* Enablenon-critical exceptions */
	Xil_ExceptionEnable();
	return XST_SUCCESS;
}
/**
 *
 * 1588中断的处理函数
 */
void deal_1588_interrupt(void*InstancePtr){
	//进入中断之后先将中断关闭
	u32 timeArray[4];
	Time_1588 time_1588;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Disable(IntcInstancePtr,crossbar_1588_interruptID);
	///////////////////////////////////////////////////
	//来中断的时候,需要读取接收寄存器的数据
#if 0//作为主时钟的时候,主要是解析Delay_Req
	switch(state){
		case 0 :
				state = 1;//开始发送FollowUp
			break;
		case 1://这是发送完FollowUP产生的中断
			state = 2;//证明Followup发送完成
			break;
		case 2:
			//接收Delayreq
	    	rx_get_128(timeArray);
	    	time_1588 = transto1588(timeArray);
	    	pack1588_to_Delay_Resp(&time_1588,&delay_resp);
	    	state = 3;
			break;
		case 3:break;


	}



#else //次时钟
	switch(state){

		case -1:
				rx_get_128(timeArray);
				sec_time_1588[1] = transto1588(timeArray);//获取t2
				LogInfo(&sec_time_1588[1]);
				state = 0;

				break;
		case 0 :
		    	rx_get_128(timeArray);
		    	sec_time_1588[0] = transto1588(timeArray);//获取t1
		    	LogInfo(&sec_time_1588[0]);
		    	state = 1;//
			break;
		case 1://这是发送完delay_req产生的中断
			state = 2;
	    	tx_get_128(timeArray);
	    	sec_time_1588[2] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[2]);
			//证明delay_req发送完成,读取发送的时间t3
			break;
		case 2:
			//接收DelayResq
	    	rx_get_128(timeArray);
	    	sec_time_1588[3] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[3]);
	    	state = -1;
			break;



	}
	printf("zhongduan = %d \n",state);


#endif


    //////////////////////////////////////////////////////
	//完成中断，再将中断使能
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);

}


/*
 * crossbar_interrupt.c
 *
 *  Created on: 2015-11-13
 *      Author: Xaxdus
 */

#include"xil_io.h"
#include"xstatus.h"
#include "crossbar_interrupt.h"
#include "crossbar_1588.h"

static INTC Intc;//全局的INTC
extern Delay_Resp delay_resp;
extern u32 interruptflag;
extern  int state;
extern Time_1588 sec_time_1588[4];
/**
 * 	初始化中断配置
 */
int init_1588_Interrupt(){
	int result;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Config* IntcConfig;
	/* Initializethe interrupt controller driver so that it is ready to
	* use.
	* */
	IntcConfig =XScuGic_LookupConfig(INTC_DEVICE_ID);
	if (IntcConfig== NULL)
	{
		return XST_FAILURE;
	}
	/* Initializethe SCU and GIC to enable the desired interrupt
	* configuration.*/
	result =XScuGic_CfgInitialize(IntcInstancePtr, IntcConfig,
	IntcConfig->CpuBaseAddress);
	if (result !=XST_SUCCESS)
	{
		return XST_FAILURE;
	}
	XScuGic_SetPriorityTriggerType(IntcInstancePtr,crossbar_1588_interruptID,
	0xA0, 0x3);
	/* Connect the interrupt handler that will be called when an
	* interrupt occurs for the device. */
	result =XScuGic_Connect(IntcInstancePtr, crossbar_1588_interruptID,
	(Xil_ExceptionHandler)deal_1588_interrupt, 0);
	if (result !=XST_SUCCESS)
	{
		return result;
	}
	/* Enable the interrupt for the 1588 controller device. */
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);
	/* Initializethe exception table and register the interrupt controller
	* handler withthe exception table. */
	Xil_ExceptionInit();
	Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,
	(Xil_ExceptionHandler)INTC_HANDLER,IntcInstancePtr);
	/* Enablenon-critical exceptions */
	Xil_ExceptionEnable();
	return XST_SUCCESS;
}
/**
 *
 * 1588中断的处理函数
 */
void deal_1588_interrupt(void*InstancePtr){
	//进入中断之后先将中断关闭
	u32 timeArray[4];
	Time_1588 time_1588;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Disable(IntcInstancePtr,crossbar_1588_interruptID);
	///////////////////////////////////////////////////
	//来中断的时候,需要读取接收寄存器的数据
#if 0//作为主时钟的时候,主要是解析Delay_Req
	switch(state){
		case 0 :
				state = 1;//开始发送FollowUp
			break;
		case 1://这是发送完FollowUP产生的中断
			state = 2;//证明Followup发送完成
			break;
		case 2:
			//接收Delayreq
	    	rx_get_128(timeArray);
	    	time_1588 = transto1588(timeArray);
	    	pack1588_to_Delay_Resp(&time_1588,&delay_resp);
	    	state = 3;
			break;
		case 3:break;


	}



#else //次时钟
	switch(state){

		case -1:
				rx_get_128(timeArray);
				sec_time_1588[1] = transto1588(timeArray);//获取t2
				LogInfo(&sec_time_1588[1]);
				state = 0;

				break;
		case 0 :
		    	rx_get_128(timeArray);
		    	sec_time_1588[0] = transto1588(timeArray);//获取t1
		    	LogInfo(&sec_time_1588[0]);
		    	state = 1;//
			break;
		case 1://这是发送完delay_req产生的中断
			state = 2;
	    	tx_get_128(timeArray);
	    	sec_time_1588[2] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[2]);
			//证明delay_req发送完成,读取发送的时间t3
			break;
		case 2:
			//接收DelayResq
	    	rx_get_128(timeArray);
	    	sec_time_1588[3] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[3]);
	    	state = -1;
			break;



	}
	printf("zhongduan = %d \n",state);


#endif


    //////////////////////////////////////////////////////
	//完成中断，再将中断使能
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);

}


/*
 * crossbar_interrupt.c
 *
 *  Created on: 2015-11-13
 *      Author: Xaxdus
 */

#include"xil_io.h"
#include"xstatus.h"
#include "crossbar_interrupt.h"
#include "crossbar_1588.h"

static INTC Intc;//全局的INTC
extern Delay_Resp delay_resp;
extern u32 interruptflag;
extern  int state;
extern Time_1588 sec_time_1588[4];
/**
 * 	初始化中断配置
 */
int init_1588_Interrupt(){
	int result;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Config* IntcConfig;
	/* Initializethe interrupt controller driver so that it is ready to
	* use.
	* */
	IntcConfig =XScuGic_LookupConfig(INTC_DEVICE_ID);
	if (IntcConfig== NULL)
	{
		return XST_FAILURE;
	}
	/* Initializethe SCU and GIC to enable the desired interrupt
	* configuration.*/
	result =XScuGic_CfgInitialize(IntcInstancePtr, IntcConfig,
	IntcConfig->CpuBaseAddress);
	if (result !=XST_SUCCESS)
	{
		return XST_FAILURE;
	}
	XScuGic_SetPriorityTriggerType(IntcInstancePtr,crossbar_1588_interruptID,
	0xA0, 0x3);
	/* Connect the interrupt handler that will be called when an
	* interrupt occurs for the device. */
	result =XScuGic_Connect(IntcInstancePtr, crossbar_1588_interruptID,
	(Xil_ExceptionHandler)deal_1588_interrupt, 0);
	if (result !=XST_SUCCESS)
	{
		return result;
	}
	/* Enable the interrupt for the 1588 controller device. */
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);
	/* Initializethe exception table and register the interrupt controller
	* handler withthe exception table. */
	Xil_ExceptionInit();
	Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,
	(Xil_ExceptionHandler)INTC_HANDLER,IntcInstancePtr);
	/* Enablenon-critical exceptions */
	Xil_ExceptionEnable();
	return XST_SUCCESS;
}
/**
 *
 * 1588中断的处理函数
 */
void deal_1588_interrupt(void*InstancePtr){
	//进入中断之后先将中断关闭
	u32 timeArray[4];
	Time_1588 time_1588;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Disable(IntcInstancePtr,crossbar_1588_interruptID);
	///////////////////////////////////////////////////
	//来中断的时候,需要读取接收寄存器的数据
#if 0//作为主时钟的时候,主要是解析Delay_Req
	switch(state){
		case 0 :
				state = 1;//开始发送FollowUp
			break;
		case 1://这是发送完FollowUP产生的中断
			state = 2;//证明Followup发送完成
			break;
		case 2:
			//接收Delayreq
	    	rx_get_128(timeArray);
	    	time_1588 = transto1588(timeArray);
	    	pack1588_to_Delay_Resp(&time_1588,&delay_resp);
	    	state = 3;
			break;
		case 3:break;


	}



#else //次时钟
	switch(state){

		case -1:
				rx_get_128(timeArray);
				sec_time_1588[1] = transto1588(timeArray);//获取t2
				LogInfo(&sec_time_1588[1]);
				state = 0;

				break;
		case 0 :
		    	rx_get_128(timeArray);
		    	sec_time_1588[0] = transto1588(timeArray);//获取t1
		    	LogInfo(&sec_time_1588[0]);
		    	state = 1;//
			break;
		case 1://这是发送完delay_req产生的中断
			state = 2;
	    	tx_get_128(timeArray);
	    	sec_time_1588[2] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[2]);
			//证明delay_req发送完成,读取发送的时间t3
			break;
		case 2:
			//接收DelayResq
	    	rx_get_128(timeArray);
	    	sec_time_1588[3] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[3]);
	    	state = -1;
			break;



	}
	printf("zhongduan = %d \n",state);


#endif


    //////////////////////////////////////////////////////
	//完成中断，再将中断使能
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);

}


/*
 * crossbar_interrupt.c
 *
 *  Created on: 2015-11-13
 *      Author: Xaxdus
 */

#include"xil_io.h"
#include"xstatus.h"
#include "crossbar_interrupt.h"
#include "crossbar_1588.h"

static INTC Intc;//全局的INTC
extern Delay_Resp delay_resp;
extern u32 interruptflag;
extern  int state;
extern Time_1588 sec_time_1588[4];
/**
 * 	初始化中断配置
 */
int init_1588_Interrupt(){
	int result;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Config* IntcConfig;
	/* Initializethe interrupt controller driver so that it is ready to
	* use.
	* */
	IntcConfig =XScuGic_LookupConfig(INTC_DEVICE_ID);
	if (IntcConfig== NULL)
	{
		return XST_FAILURE;
	}
	/* Initializethe SCU and GIC to enable the desired interrupt
	* configuration.*/
	result =XScuGic_CfgInitialize(IntcInstancePtr, IntcConfig,
	IntcConfig->CpuBaseAddress);
	if (result !=XST_SUCCESS)
	{
		return XST_FAILURE;
	}
	XScuGic_SetPriorityTriggerType(IntcInstancePtr,crossbar_1588_interruptID,
	0xA0, 0x3);
	/* Connect the interrupt handler that will be called when an
	* interrupt occurs for the device. */
	result =XScuGic_Connect(IntcInstancePtr, crossbar_1588_interruptID,
	(Xil_ExceptionHandler)deal_1588_interrupt, 0);
	if (result !=XST_SUCCESS)
	{
		return result;
	}
	/* Enable the interrupt for the 1588 controller device. */
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);
	/* Initializethe exception table and register the interrupt controller
	* handler withthe exception table. */
	Xil_ExceptionInit();
	Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,
	(Xil_ExceptionHandler)INTC_HANDLER,IntcInstancePtr);
	/* Enablenon-critical exceptions */
	Xil_ExceptionEnable();
	return XST_SUCCESS;
}
/**
 *
 * 1588中断的处理函数
 */
void deal_1588_interrupt(void*InstancePtr){
	//进入中断之后先将中断关闭
	u32 timeArray[4];
	Time_1588 time_1588;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Disable(IntcInstancePtr,crossbar_1588_interruptID);
	///////////////////////////////////////////////////
	//来中断的时候,需要读取接收寄存器的数据
#if 0//作为主时钟的时候,主要是解析Delay_Req
	switch(state){
		case 0 :
				state = 1;//开始发送FollowUp
			break;
		case 1://这是发送完FollowUP产生的中断
			state = 2;//证明Followup发送完成
			break;
		case 2:
			//接收Delayreq
	    	rx_get_128(timeArray);
	    	time_1588 = transto1588(timeArray);
	    	pack1588_to_Delay_Resp(&time_1588,&delay_resp);
	    	state = 3;
			break;
		case 3:break;


	}



#else //次时钟
	switch(state){

		case -1:
				rx_get_128(timeArray);
				sec_time_1588[1] = transto1588(timeArray);//获取t2
				LogInfo(&sec_time_1588[1]);
				state = 0;

				break;
		case 0 :
		    	rx_get_128(timeArray);
		    	sec_time_1588[0] = transto1588(timeArray);//获取t1
		    	LogInfo(&sec_time_1588[0]);
		    	state = 1;//
			break;
		case 1://这是发送完delay_req产生的中断
			state = 2;
	    	tx_get_128(timeArray);
	    	sec_time_1588[2] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[2]);
			//证明delay_req发送完成,读取发送的时间t3
			break;
		case 2:
			//接收DelayResq
	    	rx_get_128(timeArray);
	    	sec_time_1588[3] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[3]);
	    	state = -1;
			break;



	}
	printf("zhongduan = %d \n",state);


#endif


    //////////////////////////////////////////////////////
	//完成中断，再将中断使能
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);

}


/*
 * crossbar_interrupt.c
 *
 *  Created on: 2015-11-13
 *      Author: Xaxdus
 */

#include"xil_io.h"
#include"xstatus.h"
#include "crossbar_interrupt.h"
#include "crossbar_1588.h"

static INTC Intc;//全局的INTC
extern Delay_Resp delay_resp;
extern u32 interruptflag;
extern  int state;
extern Time_1588 sec_time_1588[4];
/**
 * 	初始化中断配置
 */
int init_1588_Interrupt(){
	int result;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Config* IntcConfig;
	/* Initializethe interrupt controller driver so that it is ready to
	* use.
	* */
	IntcConfig =XScuGic_LookupConfig(INTC_DEVICE_ID);
	if (IntcConfig== NULL)
	{
		return XST_FAILURE;
	}
	/* Initializethe SCU and GIC to enable the desired interrupt
	* configuration.*/
	result =XScuGic_CfgInitialize(IntcInstancePtr, IntcConfig,
	IntcConfig->CpuBaseAddress);
	if (result !=XST_SUCCESS)
	{
		return XST_FAILURE;
	}
	XScuGic_SetPriorityTriggerType(IntcInstancePtr,crossbar_1588_interruptID,
	0xA0, 0x3);
	/* Connect the interrupt handler that will be called when an
	* interrupt occurs for the device. */
	result =XScuGic_Connect(IntcInstancePtr, crossbar_1588_interruptID,
	(Xil_ExceptionHandler)deal_1588_interrupt, 0);
	if (result !=XST_SUCCESS)
	{
		return result;
	}
	/* Enable the interrupt for the 1588 controller device. */
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);
	/* Initializethe exception table and register the interrupt controller
	* handler withthe exception table. */
	Xil_ExceptionInit();
	Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,
	(Xil_ExceptionHandler)INTC_HANDLER,IntcInstancePtr);
	/* Enablenon-critical exceptions */
	Xil_ExceptionEnable();
	return XST_SUCCESS;
}
/**
 *
 * 1588中断的处理函数
 */
void deal_1588_interrupt(void*InstancePtr){
	//进入中断之后先将中断关闭
	u32 timeArray[4];
	Time_1588 time_1588;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Disable(IntcInstancePtr,crossbar_1588_interruptID);
	///////////////////////////////////////////////////
	//来中断的时候,需要读取接收寄存器的数据
#if 0//作为主时钟的时候,主要是解析Delay_Req
	switch(state){
		case 0 :
				state = 1;//开始发送FollowUp
			break;
		case 1://这是发送完FollowUP产生的中断
			state = 2;//证明Followup发送完成
			break;
		case 2:
			//接收Delayreq
	    	rx_get_128(timeArray);
	    	time_1588 = transto1588(timeArray);
	    	pack1588_to_Delay_Resp(&time_1588,&delay_resp);
	    	state = 3;
			break;
		case 3:break;


	}



#else //次时钟
	switch(state){

		case -1:
				rx_get_128(timeArray);
				sec_time_1588[1] = transto1588(timeArray);//获取t2
				LogInfo(&sec_time_1588[1]);
				state = 0;

				break;
		case 0 :
		    	rx_get_128(timeArray);
		    	sec_time_1588[0] = transto1588(timeArray);//获取t1
		    	LogInfo(&sec_time_1588[0]);
		    	state = 1;//
			break;
		case 1://这是发送完delay_req产生的中断
			state = 2;
	    	tx_get_128(timeArray);
	    	sec_time_1588[2] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[2]);
			//证明delay_req发送完成,读取发送的时间t3
			break;
		case 2:
			//接收DelayResq
	    	rx_get_128(timeArray);
	    	sec_time_1588[3] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[3]);
	    	state = -1;
			break;



	}
	printf("zhongduan = %d \n",state);


#endif


    //////////////////////////////////////////////////////
	//完成中断，再将中断使能
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);

}


/*
 * crossbar_interrupt.c
 *
 *  Created on: 2015-11-13
 *      Author: Xaxdus
 */

#include"xil_io.h"
#include"xstatus.h"
#include "crossbar_interrupt.h"
#include "crossbar_1588.h"

static INTC Intc;//全局的INTC
extern Delay_Resp delay_resp;
extern u32 interruptflag;
extern  int state;
extern Time_1588 sec_time_1588[4];
/**
 * 	初始化中断配置
 */
int init_1588_Interrupt(){
	int result;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Config* IntcConfig;
	/* Initializethe interrupt controller driver so that it is ready to
	* use.
	* */
	IntcConfig =XScuGic_LookupConfig(INTC_DEVICE_ID);
	if (IntcConfig== NULL)
	{
		return XST_FAILURE;
	}
	/* Initializethe SCU and GIC to enable the desired interrupt
	* configuration.*/
	result =XScuGic_CfgInitialize(IntcInstancePtr, IntcConfig,
	IntcConfig->CpuBaseAddress);
	if (result !=XST_SUCCESS)
	{
		return XST_FAILURE;
	}
	XScuGic_SetPriorityTriggerType(IntcInstancePtr,crossbar_1588_interruptID,
	0xA0, 0x3);
	/* Connect the interrupt handler that will be called when an
	* interrupt occurs for the device. */
	result =XScuGic_Connect(IntcInstancePtr, crossbar_1588_interruptID,
	(Xil_ExceptionHandler)deal_1588_interrupt, 0);
	if (result !=XST_SUCCESS)
	{
		return result;
	}
	/* Enable the interrupt for the 1588 controller device. */
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);
	/* Initializethe exception table and register the interrupt controller
	* handler withthe exception table. */
	Xil_ExceptionInit();
	Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,
	(Xil_ExceptionHandler)INTC_HANDLER,IntcInstancePtr);
	/* Enablenon-critical exceptions */
	Xil_ExceptionEnable();
	return XST_SUCCESS;
}
/**
 *
 * 1588中断的处理函数
 */
void deal_1588_interrupt(void*InstancePtr){
	//进入中断之后先将中断关闭
	u32 timeArray[4];
	Time_1588 time_1588;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Disable(IntcInstancePtr,crossbar_1588_interruptID);
	///////////////////////////////////////////////////
	//来中断的时候,需要读取接收寄存器的数据
#if 0//作为主时钟的时候,主要是解析Delay_Req
	switch(state){
		case 0 :
				state = 1;//开始发送FollowUp
			break;
		case 1://这是发送完FollowUP产生的中断
			state = 2;//证明Followup发送完成
			break;
		case 2:
			//接收Delayreq
	    	rx_get_128(timeArray);
	    	time_1588 = transto1588(timeArray);
	    	pack1588_to_Delay_Resp(&time_1588,&delay_resp);
	    	state = 3;
			break;
		case 3:break;


	}



#else //次时钟
	switch(state){

		case -1:
				rx_get_128(timeArray);
				sec_time_1588[1] = transto1588(timeArray);//获取t2
				LogInfo(&sec_time_1588[1]);
				state = 0;

				break;
		case 0 :
		    	rx_get_128(timeArray);
		    	sec_time_1588[0] = transto1588(timeArray);//获取t1
		    	LogInfo(&sec_time_1588[0]);
		    	state = 1;//
			break;
		case 1://这是发送完delay_req产生的中断
			state = 2;
	    	tx_get_128(timeArray);
	    	sec_time_1588[2] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[2]);
			//证明delay_req发送完成,读取发送的时间t3
			break;
		case 2:
			//接收DelayResq
	    	rx_get_128(timeArray);
	    	sec_time_1588[3] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[3]);
	    	state = -1;
			break;



	}
	printf("zhongduan = %d \n",state);


#endif


    //////////////////////////////////////////////////////
	//完成中断，再将中断使能
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);

}


/*
 * crossbar_interrupt.c
 *
 *  Created on: 2015-11-13
 *      Author: Xaxdus
 */

#include"xil_io.h"
#include"xstatus.h"
#include "crossbar_interrupt.h"
#include "crossbar_1588.h"

static INTC Intc;//全局的INTC
extern Delay_Resp delay_resp;
extern u32 interruptflag;
extern  int state;
extern Time_1588 sec_time_1588[4];
/**
 * 	初始化中断配置
 */
int init_1588_Interrupt(){
	int result;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Config* IntcConfig;
	/* Initializethe interrupt controller driver so that it is ready to
	* use.
	* */
	IntcConfig =XScuGic_LookupConfig(INTC_DEVICE_ID);
	if (IntcConfig== NULL)
	{
		return XST_FAILURE;
	}
	/* Initializethe SCU and GIC to enable the desired interrupt
	* configuration.*/
	result =XScuGic_CfgInitialize(IntcInstancePtr, IntcConfig,
	IntcConfig->CpuBaseAddress);
	if (result !=XST_SUCCESS)
	{
		return XST_FAILURE;
	}
	XScuGic_SetPriorityTriggerType(IntcInstancePtr,crossbar_1588_interruptID,
	0xA0, 0x3);
	/* Connect the interrupt handler that will be called when an
	* interrupt occurs for the device. */
	result =XScuGic_Connect(IntcInstancePtr, crossbar_1588_interruptID,
	(Xil_ExceptionHandler)deal_1588_interrupt, 0);
	if (result !=XST_SUCCESS)
	{
		return result;
	}
	/* Enable the interrupt for the 1588 controller device. */
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);
	/* Initializethe exception table and register the interrupt controller
	* handler withthe exception table. */
	Xil_ExceptionInit();
	Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,
	(Xil_ExceptionHandler)INTC_HANDLER,IntcInstancePtr);
	/* Enablenon-critical exceptions */
	Xil_ExceptionEnable();
	return XST_SUCCESS;
}
/**
 *
 * 1588中断的处理函数
 */
void deal_1588_interrupt(void*InstancePtr){
	//进入中断之后先将中断关闭
	u32 timeArray[4];
	Time_1588 time_1588;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Disable(IntcInstancePtr,crossbar_1588_interruptID);
	///////////////////////////////////////////////////
	//来中断的时候,需要读取接收寄存器的数据
#if 0//作为主时钟的时候,主要是解析Delay_Req
	switch(state){
		case 0 :
				state = 1;//开始发送FollowUp
			break;
		case 1://这是发送完FollowUP产生的中断
			state = 2;//证明Followup发送完成
			break;
		case 2:
			//接收Delayreq
	    	rx_get_128(timeArray);
	    	time_1588 = transto1588(timeArray);
	    	pack1588_to_Delay_Resp(&time_1588,&delay_resp);
	    	state = 3;
			break;
		case 3:break;


	}



#else //次时钟
	switch(state){

		case -1:
				rx_get_128(timeArray);
				sec_time_1588[1] = transto1588(timeArray);//获取t2
				LogInfo(&sec_time_1588[1]);
				state = 0;

				break;
		case 0 :
		    	rx_get_128(timeArray);
		    	sec_time_1588[0] = transto1588(timeArray);//获取t1
		    	LogInfo(&sec_time_1588[0]);
		    	state = 1;//
			break;
		case 1://这是发送完delay_req产生的中断
			state = 2;
	    	tx_get_128(timeArray);
	    	sec_time_1588[2] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[2]);
			//证明delay_req发送完成,读取发送的时间t3
			break;
		case 2:
			//接收DelayResq
	    	rx_get_128(timeArray);
	    	sec_time_1588[3] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[3]);
	    	state = -1;
			break;



	}
	printf("zhongduan = %d \n",state);


#endif


    //////////////////////////////////////////////////////
	//完成中断，再将中断使能
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);

}


/*
 * crossbar_interrupt.c
 *
 *  Created on: 2015-11-13
 *      Author: Xaxdus
 */

#include"xil_io.h"
#include"xstatus.h"
#include "crossbar_interrupt.h"
#include "crossbar_1588.h"

static INTC Intc;//全局的INTC
extern Delay_Resp delay_resp;
extern u32 interruptflag;
extern  int state;
extern Time_1588 sec_time_1588[4];
/**
 * 	初始化中断配置
 */
int init_1588_Interrupt(){
	int result;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Config* IntcConfig;
	/* Initializethe interrupt controller driver so that it is ready to
	* use.
	* */
	IntcConfig =XScuGic_LookupConfig(INTC_DEVICE_ID);
	if (IntcConfig== NULL)
	{
		return XST_FAILURE;
	}
	/* Initializethe SCU and GIC to enable the desired interrupt
	* configuration.*/
	result =XScuGic_CfgInitialize(IntcInstancePtr, IntcConfig,
	IntcConfig->CpuBaseAddress);
	if (result !=XST_SUCCESS)
	{
		return XST_FAILURE;
	}
	XScuGic_SetPriorityTriggerType(IntcInstancePtr,crossbar_1588_interruptID,
	0xA0, 0x3);
	/* Connect the interrupt handler that will be called when an
	* interrupt occurs for the device. */
	result =XScuGic_Connect(IntcInstancePtr, crossbar_1588_interruptID,
	(Xil_ExceptionHandler)deal_1588_interrupt, 0);
	if (result !=XST_SUCCESS)
	{
		return result;
	}
	/* Enable the interrupt for the 1588 controller device. */
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);
	/* Initializethe exception table and register the interrupt controller
	* handler withthe exception table. */
	Xil_ExceptionInit();
	Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,
	(Xil_ExceptionHandler)INTC_HANDLER,IntcInstancePtr);
	/* Enablenon-critical exceptions */
	Xil_ExceptionEnable();
	return XST_SUCCESS;
}
/**
 *
 * 1588中断的处理函数
 */
void deal_1588_interrupt(void*InstancePtr){
	//进入中断之后先将中断关闭
	u32 timeArray[4];
	Time_1588 time_1588;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Disable(IntcInstancePtr,crossbar_1588_interruptID);
	///////////////////////////////////////////////////
	//来中断的时候,需要读取接收寄存器的数据
#if 0//作为主时钟的时候,主要是解析Delay_Req
	switch(state){
		case 0 :
				state = 1;//开始发送FollowUp
			break;
		case 1://这是发送完FollowUP产生的中断
			state = 2;//证明Followup发送完成
			break;
		case 2:
			//接收Delayreq
	    	rx_get_128(timeArray);
	    	time_1588 = transto1588(timeArray);
	    	pack1588_to_Delay_Resp(&time_1588,&delay_resp);
	    	state = 3;
			break;
		case 3:break;


	}



#else //次时钟
	switch(state){

		case -1:
				rx_get_128(timeArray);
				sec_time_1588[1] = transto1588(timeArray);//获取t2
				LogInfo(&sec_time_1588[1]);
				state = 0;

				break;
		case 0 :
		    	rx_get_128(timeArray);
		    	sec_time_1588[0] = transto1588(timeArray);//获取t1
		    	LogInfo(&sec_time_1588[0]);
		    	state = 1;//
			break;
		case 1://这是发送完delay_req产生的中断
			state = 2;
	    	tx_get_128(timeArray);
	    	sec_time_1588[2] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[2]);
			//证明delay_req发送完成,读取发送的时间t3
			break;
		case 2:
			//接收DelayResq
	    	rx_get_128(timeArray);
	    	sec_time_1588[3] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[3]);
	    	state = -1;
			break;



	}
	printf("zhongduan = %d \n",state);


#endif


    //////////////////////////////////////////////////////
	//完成中断，再将中断使能
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);

}


/*
 * crossbar_interrupt.c
 *
 *  Created on: 2015-11-13
 *      Author: Xaxdus
 */

#include"xil_io.h"
#include"xstatus.h"
#include "crossbar_interrupt.h"
#include "crossbar_1588.h"

static INTC Intc;//全局的INTC
extern Delay_Resp delay_resp;
extern u32 interruptflag;
extern  int state;
extern Time_1588 sec_time_1588[4];
/**
 * 	初始化中断配置
 */
int init_1588_Interrupt(){
	int result;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Config* IntcConfig;
	/* Initializethe interrupt controller driver so that it is ready to
	* use.
	* */
	IntcConfig =XScuGic_LookupConfig(INTC_DEVICE_ID);
	if (IntcConfig== NULL)
	{
		return XST_FAILURE;
	}
	/* Initializethe SCU and GIC to enable the desired interrupt
	* configuration.*/
	result =XScuGic_CfgInitialize(IntcInstancePtr, IntcConfig,
	IntcConfig->CpuBaseAddress);
	if (result !=XST_SUCCESS)
	{
		return XST_FAILURE;
	}
	XScuGic_SetPriorityTriggerType(IntcInstancePtr,crossbar_1588_interruptID,
	0xA0, 0x3);
	/* Connect the interrupt handler that will be called when an
	* interrupt occurs for the device. */
	result =XScuGic_Connect(IntcInstancePtr, crossbar_1588_interruptID,
	(Xil_ExceptionHandler)deal_1588_interrupt, 0);
	if (result !=XST_SUCCESS)
	{
		return result;
	}
	/* Enable the interrupt for the 1588 controller device. */
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);
	/* Initializethe exception table and register the interrupt controller
	* handler withthe exception table. */
	Xil_ExceptionInit();
	Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,
	(Xil_ExceptionHandler)INTC_HANDLER,IntcInstancePtr);
	/* Enablenon-critical exceptions */
	Xil_ExceptionEnable();
	return XST_SUCCESS;
}
/**
 *
 * 1588中断的处理函数
 */
void deal_1588_interrupt(void*InstancePtr){
	//进入中断之后先将中断关闭
	u32 timeArray[4];
	Time_1588 time_1588;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Disable(IntcInstancePtr,crossbar_1588_interruptID);
	///////////////////////////////////////////////////
	//来中断的时候,需要读取接收寄存器的数据
#if 0//作为主时钟的时候,主要是解析Delay_Req
	switch(state){
		case 0 :
				state = 1;//开始发送FollowUp
			break;
		case 1://这是发送完FollowUP产生的中断
			state = 2;//证明Followup发送完成
			break;
		case 2:
			//接收Delayreq
	    	rx_get_128(timeArray);
	    	time_1588 = transto1588(timeArray);
	    	pack1588_to_Delay_Resp(&time_1588,&delay_resp);
	    	state = 3;
			break;
		case 3:break;


	}



#else //次时钟
	switch(state){

		case -1:
				rx_get_128(timeArray);
				sec_time_1588[1] = transto1588(timeArray);//获取t2
				LogInfo(&sec_time_1588[1]);
				state = 0;

				break;
		case 0 :
		    	rx_get_128(timeArray);
		    	sec_time_1588[0] = transto1588(timeArray);//获取t1
		    	LogInfo(&sec_time_1588[0]);
		    	state = 1;//
			break;
		case 1://这是发送完delay_req产生的中断
			state = 2;
	    	tx_get_128(timeArray);
	    	sec_time_1588[2] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[2]);
			//证明delay_req发送完成,读取发送的时间t3
			break;
		case 2:
			//接收DelayResq
	    	rx_get_128(timeArray);
	    	sec_time_1588[3] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[3]);
	    	state = -1;
			break;



	}
	printf("zhongduan = %d \n",state);


#endif


    //////////////////////////////////////////////////////
	//完成中断，再将中断使能
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);

}


/*
 * crossbar_interrupt.c
 *
 *  Created on: 2015-11-13
 *      Author: Xaxdus
 */

#include"xil_io.h"
#include"xstatus.h"
#include "crossbar_interrupt.h"
#include "crossbar_1588.h"

static INTC Intc;//全局的INTC
extern Delay_Resp delay_resp;
extern u32 interruptflag;
extern  int state;
extern Time_1588 sec_time_1588[4];
/**
 * 	初始化中断配置
 */
int init_1588_Interrupt(){
	int result;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Config* IntcConfig;
	/* Initializethe interrupt controller driver so that it is ready to
	* use.
	* */
	IntcConfig =XScuGic_LookupConfig(INTC_DEVICE_ID);
	if (IntcConfig== NULL)
	{
		return XST_FAILURE;
	}
	/* Initializethe SCU and GIC to enable the desired interrupt
	* configuration.*/
	result =XScuGic_CfgInitialize(IntcInstancePtr, IntcConfig,
	IntcConfig->CpuBaseAddress);
	if (result !=XST_SUCCESS)
	{
		return XST_FAILURE;
	}
	XScuGic_SetPriorityTriggerType(IntcInstancePtr,crossbar_1588_interruptID,
	0xA0, 0x3);
	/* Connect the interrupt handler that will be called when an
	* interrupt occurs for the device. */
	result =XScuGic_Connect(IntcInstancePtr, crossbar_1588_interruptID,
	(Xil_ExceptionHandler)deal_1588_interrupt, 0);
	if (result !=XST_SUCCESS)
	{
		return result;
	}
	/* Enable the interrupt for the 1588 controller device. */
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);
	/* Initializethe exception table and register the interrupt controller
	* handler withthe exception table. */
	Xil_ExceptionInit();
	Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,
	(Xil_ExceptionHandler)INTC_HANDLER,IntcInstancePtr);
	/* Enablenon-critical exceptions */
	Xil_ExceptionEnable();
	return XST_SUCCESS;
}
/**
 *
 * 1588中断的处理函数
 */
void deal_1588_interrupt(void*InstancePtr){
	//进入中断之后先将中断关闭
	u32 timeArray[4];
	Time_1588 time_1588;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Disable(IntcInstancePtr,crossbar_1588_interruptID);
	///////////////////////////////////////////////////
	//来中断的时候,需要读取接收寄存器的数据
#if 0//作为主时钟的时候,主要是解析Delay_Req
	switch(state){
		case 0 :
				state = 1;//开始发送FollowUp
			break;
		case 1://这是发送完FollowUP产生的中断
			state = 2;//证明Followup发送完成
			break;
		case 2:
			//接收Delayreq
	    	rx_get_128(timeArray);
	    	time_1588 = transto1588(timeArray);
	    	pack1588_to_Delay_Resp(&time_1588,&delay_resp);
	    	state = 3;
			break;
		case 3:break;


	}



#else //次时钟
	switch(state){

		case -1:
				rx_get_128(timeArray);
				sec_time_1588[1] = transto1588(timeArray);//获取t2
				LogInfo(&sec_time_1588[1]);
				state = 0;

				break;
		case 0 :
		    	rx_get_128(timeArray);
		    	sec_time_1588[0] = transto1588(timeArray);//获取t1
		    	LogInfo(&sec_time_1588[0]);
		    	state = 1;//
			break;
		case 1://这是发送完delay_req产生的中断
			state = 2;
	    	tx_get_128(timeArray);
	    	sec_time_1588[2] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[2]);
			//证明delay_req发送完成,读取发送的时间t3
			break;
		case 2:
			//接收DelayResq
	    	rx_get_128(timeArray);
	    	sec_time_1588[3] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[3]);
	    	state = -1;
			break;



	}
	printf("zhongduan = %d \n",state);


#endif


    //////////////////////////////////////////////////////
	//完成中断，再将中断使能
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);

}


/*
 * crossbar_interrupt.c
 *
 *  Created on: 2015-11-13
 *      Author: Xaxdus
 */

#include"xil_io.h"
#include"xstatus.h"
#include "crossbar_interrupt.h"
#include "crossbar_1588.h"

static INTC Intc;//全局的INTC
extern Delay_Resp delay_resp;
extern u32 interruptflag;
extern  int state;
extern Time_1588 sec_time_1588[4];
/**
 * 	初始化中断配置
 */
int init_1588_Interrupt(){
	int result;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Config* IntcConfig;
	/* Initializethe interrupt controller driver so that it is ready to
	* use.
	* */
	IntcConfig =XScuGic_LookupConfig(INTC_DEVICE_ID);
	if (IntcConfig== NULL)
	{
		return XST_FAILURE;
	}
	/* Initializethe SCU and GIC to enable the desired interrupt
	* configuration.*/
	result =XScuGic_CfgInitialize(IntcInstancePtr, IntcConfig,
	IntcConfig->CpuBaseAddress);
	if (result !=XST_SUCCESS)
	{
		return XST_FAILURE;
	}
	XScuGic_SetPriorityTriggerType(IntcInstancePtr,crossbar_1588_interruptID,
	0xA0, 0x3);
	/* Connect the interrupt handler that will be called when an
	* interrupt occurs for the device. */
	result =XScuGic_Connect(IntcInstancePtr, crossbar_1588_interruptID,
	(Xil_ExceptionHandler)deal_1588_interrupt, 0);
	if (result !=XST_SUCCESS)
	{
		return result;
	}
	/* Enable the interrupt for the 1588 controller device. */
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);
	/* Initializethe exception table and register the interrupt controller
	* handler withthe exception table. */
	Xil_ExceptionInit();
	Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,
	(Xil_ExceptionHandler)INTC_HANDLER,IntcInstancePtr);
	/* Enablenon-critical exceptions */
	Xil_ExceptionEnable();
	return XST_SUCCESS;
}
/**
 *
 * 1588中断的处理函数
 */
void deal_1588_interrupt(void*InstancePtr){
	//进入中断之后先将中断关闭
	u32 timeArray[4];
	Time_1588 time_1588;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Disable(IntcInstancePtr,crossbar_1588_interruptID);
	///////////////////////////////////////////////////
	//来中断的时候,需要读取接收寄存器的数据
#if 0//作为主时钟的时候,主要是解析Delay_Req
	switch(state){
		case 0 :
				state = 1;//开始发送FollowUp
			break;
		case 1://这是发送完FollowUP产生的中断
			state = 2;//证明Followup发送完成
			break;
		case 2:
			//接收Delayreq
	    	rx_get_128(timeArray);
	    	time_1588 = transto1588(timeArray);
	    	pack1588_to_Delay_Resp(&time_1588,&delay_resp);
	    	state = 3;
			break;
		case 3:break;


	}



#else //次时钟
	switch(state){

		case -1:
				rx_get_128(timeArray);
				sec_time_1588[1] = transto1588(timeArray);//获取t2
				LogInfo(&sec_time_1588[1]);
				state = 0;

				break;
		case 0 :
		    	rx_get_128(timeArray);
		    	sec_time_1588[0] = transto1588(timeArray);//获取t1
		    	LogInfo(&sec_time_1588[0]);
		    	state = 1;//
			break;
		case 1://这是发送完delay_req产生的中断
			state = 2;
	    	tx_get_128(timeArray);
	    	sec_time_1588[2] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[2]);
			//证明delay_req发送完成,读取发送的时间t3
			break;
		case 2:
			//接收DelayResq
	    	rx_get_128(timeArray);
	    	sec_time_1588[3] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[3]);
	    	state = -1;
			break;



	}
	printf("zhongduan = %d \n",state);


#endif


    //////////////////////////////////////////////////////
	//完成中断，再将中断使能
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);

}


/*
 * crossbar_interrupt.c
 *
 *  Created on: 2015-11-13
 *      Author: Xaxdus
 */

#include"xil_io.h"
#include"xstatus.h"
#include "crossbar_interrupt.h"
#include "crossbar_1588.h"

static INTC Intc;//全局的INTC
extern Delay_Resp delay_resp;
extern u32 interruptflag;
extern  int state;
extern Time_1588 sec_time_1588[4];
/**
 * 	初始化中断配置
 */
int init_1588_Interrupt(){
	int result;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Config* IntcConfig;
	/* Initializethe interrupt controller driver so that it is ready to
	* use.
	* */
	IntcConfig =XScuGic_LookupConfig(INTC_DEVICE_ID);
	if (IntcConfig== NULL)
	{
		return XST_FAILURE;
	}
	/* Initializethe SCU and GIC to enable the desired interrupt
	* configuration.*/
	result =XScuGic_CfgInitialize(IntcInstancePtr, IntcConfig,
	IntcConfig->CpuBaseAddress);
	if (result !=XST_SUCCESS)
	{
		return XST_FAILURE;
	}
	XScuGic_SetPriorityTriggerType(IntcInstancePtr,crossbar_1588_interruptID,
	0xA0, 0x3);
	/* Connect the interrupt handler that will be called when an
	* interrupt occurs for the device. */
	result =XScuGic_Connect(IntcInstancePtr, crossbar_1588_interruptID,
	(Xil_ExceptionHandler)deal_1588_interrupt, 0);
	if (result !=XST_SUCCESS)
	{
		return result;
	}
	/* Enable the interrupt for the 1588 controller device. */
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);
	/* Initializethe exception table and register the interrupt controller
	* handler withthe exception table. */
	Xil_ExceptionInit();
	Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,
	(Xil_ExceptionHandler)INTC_HANDLER,IntcInstancePtr);
	/* Enablenon-critical exceptions */
	Xil_ExceptionEnable();
	return XST_SUCCESS;
}
/**
 *
 * 1588中断的处理函数
 */
void deal_1588_interrupt(void*InstancePtr){
	//进入中断之后先将中断关闭
	u32 timeArray[4];
	Time_1588 time_1588;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Disable(IntcInstancePtr,crossbar_1588_interruptID);
	///////////////////////////////////////////////////
	//来中断的时候,需要读取接收寄存器的数据
#if 0//作为主时钟的时候,主要是解析Delay_Req
	switch(state){
		case 0 :
				state = 1;//开始发送FollowUp
			break;
		case 1://这是发送完FollowUP产生的中断
			state = 2;//证明Followup发送完成
			break;
		case 2:
			//接收Delayreq
	    	rx_get_128(timeArray);
	    	time_1588 = transto1588(timeArray);
	    	pack1588_to_Delay_Resp(&time_1588,&delay_resp);
	    	state = 3;
			break;
		case 3:break;


	}



#else //次时钟
	switch(state){

		case -1:
				rx_get_128(timeArray);
				sec_time_1588[1] = transto1588(timeArray);//获取t2
				LogInfo(&sec_time_1588[1]);
				state = 0;

				break;
		case 0 :
		    	rx_get_128(timeArray);
		    	sec_time_1588[0] = transto1588(timeArray);//获取t1
		    	LogInfo(&sec_time_1588[0]);
		    	state = 1;//
			break;
		case 1://这是发送完delay_req产生的中断
			state = 2;
	    	tx_get_128(timeArray);
	    	sec_time_1588[2] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[2]);
			//证明delay_req发送完成,读取发送的时间t3
			break;
		case 2:
			//接收DelayResq
	    	rx_get_128(timeArray);
	    	sec_time_1588[3] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[3]);
	    	state = -1;
			break;



	}
	printf("zhongduan = %d \n",state);


#endif


    //////////////////////////////////////////////////////
	//完成中断，再将中断使能
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);

}


/*
 * crossbar_interrupt.c
 *
 *  Created on: 2015-11-13
 *      Author: Xaxdus
 */

#include"xil_io.h"
#include"xstatus.h"
#include "crossbar_interrupt.h"
#include "crossbar_1588.h"

static INTC Intc;//全局的INTC
extern Delay_Resp delay_resp;
extern u32 interruptflag;
extern  int state;
extern Time_1588 sec_time_1588[4];
/**
 * 	初始化中断配置
 */
int init_1588_Interrupt(){
	int result;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Config* IntcConfig;
	/* Initializethe interrupt controller driver so that it is ready to
	* use.
	* */
	IntcConfig =XScuGic_LookupConfig(INTC_DEVICE_ID);
	if (IntcConfig== NULL)
	{
		return XST_FAILURE;
	}
	/* Initializethe SCU and GIC to enable the desired interrupt
	* configuration.*/
	result =XScuGic_CfgInitialize(IntcInstancePtr, IntcConfig,
	IntcConfig->CpuBaseAddress);
	if (result !=XST_SUCCESS)
	{
		return XST_FAILURE;
	}
	XScuGic_SetPriorityTriggerType(IntcInstancePtr,crossbar_1588_interruptID,
	0xA0, 0x3);
	/* Connect the interrupt handler that will be called when an
	* interrupt occurs for the device. */
	result =XScuGic_Connect(IntcInstancePtr, crossbar_1588_interruptID,
	(Xil_ExceptionHandler)deal_1588_interrupt, 0);
	if (result !=XST_SUCCESS)
	{
		return result;
	}
	/* Enable the interrupt for the 1588 controller device. */
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);
	/* Initializethe exception table and register the interrupt controller
	* handler withthe exception table. */
	Xil_ExceptionInit();
	Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,
	(Xil_ExceptionHandler)INTC_HANDLER,IntcInstancePtr);
	/* Enablenon-critical exceptions */
	Xil_ExceptionEnable();
	return XST_SUCCESS;
}
/**
 *
 * 1588中断的处理函数
 */
void deal_1588_interrupt(void*InstancePtr){
	//进入中断之后先将中断关闭
	u32 timeArray[4];
	Time_1588 time_1588;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Disable(IntcInstancePtr,crossbar_1588_interruptID);
	///////////////////////////////////////////////////
	//来中断的时候,需要读取接收寄存器的数据
#if 0//作为主时钟的时候,主要是解析Delay_Req
	switch(state){
		case 0 :
				state = 1;//开始发送FollowUp
			break;
		case 1://这是发送完FollowUP产生的中断
			state = 2;//证明Followup发送完成
			break;
		case 2:
			//接收Delayreq
	    	rx_get_128(timeArray);
	    	time_1588 = transto1588(timeArray);
	    	pack1588_to_Delay_Resp(&time_1588,&delay_resp);
	    	state = 3;
			break;
		case 3:break;


	}



#else //次时钟
	switch(state){

		case -1:
				rx_get_128(timeArray);
				sec_time_1588[1] = transto1588(timeArray);//获取t2
				LogInfo(&sec_time_1588[1]);
				state = 0;

				break;
		case 0 :
		    	rx_get_128(timeArray);
		    	sec_time_1588[0] = transto1588(timeArray);//获取t1
		    	LogInfo(&sec_time_1588[0]);
		    	state = 1;//
			break;
		case 1://这是发送完delay_req产生的中断
			state = 2;
	    	tx_get_128(timeArray);
	    	sec_time_1588[2] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[2]);
			//证明delay_req发送完成,读取发送的时间t3
			break;
		case 2:
			//接收DelayResq
	    	rx_get_128(timeArray);
	    	sec_time_1588[3] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[3]);
	    	state = -1;
			break;



	}
	printf("zhongduan = %d \n",state);


#endif


    //////////////////////////////////////////////////////
	//完成中断，再将中断使能
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);

}


/*
 * crossbar_interrupt.c
 *
 *  Created on: 2015-11-13
 *      Author: Xaxdus
 */

#include"xil_io.h"
#include"xstatus.h"
#include "crossbar_interrupt.h"
#include "crossbar_1588.h"

static INTC Intc;//全局的INTC
extern Delay_Resp delay_resp;
extern u32 interruptflag;
extern  int state;
extern Time_1588 sec_time_1588[4];
/**
 * 	初始化中断配置
 */
int init_1588_Interrupt(){
	int result;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Config* IntcConfig;
	/* Initializethe interrupt controller driver so that it is ready to
	* use.
	* */
	IntcConfig =XScuGic_LookupConfig(INTC_DEVICE_ID);
	if (IntcConfig== NULL)
	{
		return XST_FAILURE;
	}
	/* Initializethe SCU and GIC to enable the desired interrupt
	* configuration.*/
	result =XScuGic_CfgInitialize(IntcInstancePtr, IntcConfig,
	IntcConfig->CpuBaseAddress);
	if (result !=XST_SUCCESS)
	{
		return XST_FAILURE;
	}
	XScuGic_SetPriorityTriggerType(IntcInstancePtr,crossbar_1588_interruptID,
	0xA0, 0x3);
	/* Connect the interrupt handler that will be called when an
	* interrupt occurs for the device. */
	result =XScuGic_Connect(IntcInstancePtr, crossbar_1588_interruptID,
	(Xil_ExceptionHandler)deal_1588_interrupt, 0);
	if (result !=XST_SUCCESS)
	{
		return result;
	}
	/* Enable the interrupt for the 1588 controller device. */
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);
	/* Initializethe exception table and register the interrupt controller
	* handler withthe exception table. */
	Xil_ExceptionInit();
	Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,
	(Xil_ExceptionHandler)INTC_HANDLER,IntcInstancePtr);
	/* Enablenon-critical exceptions */
	Xil_ExceptionEnable();
	return XST_SUCCESS;
}
/**
 *
 * 1588中断的处理函数
 */
void deal_1588_interrupt(void*InstancePtr){
	//进入中断之后先将中断关闭
	u32 timeArray[4];
	Time_1588 time_1588;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Disable(IntcInstancePtr,crossbar_1588_interruptID);
	///////////////////////////////////////////////////
	//来中断的时候,需要读取接收寄存器的数据
#if 0//作为主时钟的时候,主要是解析Delay_Req
	switch(state){
		case 0 :
				state = 1;//开始发送FollowUp
			break;
		case 1://这是发送完FollowUP产生的中断
			state = 2;//证明Followup发送完成
			break;
		case 2:
			//接收Delayreq
	    	rx_get_128(timeArray);
	    	time_1588 = transto1588(timeArray);
	    	pack1588_to_Delay_Resp(&time_1588,&delay_resp);
	    	state = 3;
			break;
		case 3:break;


	}



#else //次时钟
	switch(state){

		case -1:
				rx_get_128(timeArray);
				sec_time_1588[1] = transto1588(timeArray);//获取t2
				LogInfo(&sec_time_1588[1]);
				state = 0;

				break;
		case 0 :
		    	rx_get_128(timeArray);
		    	sec_time_1588[0] = transto1588(timeArray);//获取t1
		    	LogInfo(&sec_time_1588[0]);
		    	state = 1;//
			break;
		case 1://这是发送完delay_req产生的中断
			state = 2;
	    	tx_get_128(timeArray);
	    	sec_time_1588[2] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[2]);
			//证明delay_req发送完成,读取发送的时间t3
			break;
		case 2:
			//接收DelayResq
	    	rx_get_128(timeArray);
	    	sec_time_1588[3] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[3]);
	    	state = -1;
			break;



	}
	printf("zhongduan = %d \n",state);


#endif


    //////////////////////////////////////////////////////
	//完成中断，再将中断使能
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);

}


/*
 * crossbar_interrupt.c
 *
 *  Created on: 2015-11-13
 *      Author: Xaxdus
 */

#include"xil_io.h"
#include"xstatus.h"
#include "crossbar_interrupt.h"
#include "crossbar_1588.h"

static INTC Intc;//全局的INTC
extern Delay_Resp delay_resp;
extern u32 interruptflag;
extern  int state;
extern Time_1588 sec_time_1588[4];
/**
 * 	初始化中断配置
 */
int init_1588_Interrupt(){
	int result;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Config* IntcConfig;
	/* Initializethe interrupt controller driver so that it is ready to
	* use.
	* */
	IntcConfig =XScuGic_LookupConfig(INTC_DEVICE_ID);
	if (IntcConfig== NULL)
	{
		return XST_FAILURE;
	}
	/* Initializethe SCU and GIC to enable the desired interrupt
	* configuration.*/
	result =XScuGic_CfgInitialize(IntcInstancePtr, IntcConfig,
	IntcConfig->CpuBaseAddress);
	if (result !=XST_SUCCESS)
	{
		return XST_FAILURE;
	}
	XScuGic_SetPriorityTriggerType(IntcInstancePtr,crossbar_1588_interruptID,
	0xA0, 0x3);
	/* Connect the interrupt handler that will be called when an
	* interrupt occurs for the device. */
	result =XScuGic_Connect(IntcInstancePtr, crossbar_1588_interruptID,
	(Xil_ExceptionHandler)deal_1588_interrupt, 0);
	if (result !=XST_SUCCESS)
	{
		return result;
	}
	/* Enable the interrupt for the 1588 controller device. */
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);
	/* Initializethe exception table and register the interrupt controller
	* handler withthe exception table. */
	Xil_ExceptionInit();
	Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,
	(Xil_ExceptionHandler)INTC_HANDLER,IntcInstancePtr);
	/* Enablenon-critical exceptions */
	Xil_ExceptionEnable();
	return XST_SUCCESS;
}
/**
 *
 * 1588中断的处理函数
 */
void deal_1588_interrupt(void*InstancePtr){
	//进入中断之后先将中断关闭
	u32 timeArray[4];
	Time_1588 time_1588;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Disable(IntcInstancePtr,crossbar_1588_interruptID);
	///////////////////////////////////////////////////
	//来中断的时候,需要读取接收寄存器的数据
#if 0//作为主时钟的时候,主要是解析Delay_Req
	switch(state){
		case 0 :
				state = 1;//开始发送FollowUp
			break;
		case 1://这是发送完FollowUP产生的中断
			state = 2;//证明Followup发送完成
			break;
		case 2:
			//接收Delayreq
	    	rx_get_128(timeArray);
	    	time_1588 = transto1588(timeArray);
	    	pack1588_to_Delay_Resp(&time_1588,&delay_resp);
	    	state = 3;
			break;
		case 3:break;


	}



#else //次时钟
	switch(state){

		case -1:
				rx_get_128(timeArray);
				sec_time_1588[1] = transto1588(timeArray);//获取t2
				LogInfo(&sec_time_1588[1]);
				state = 0;

				break;
		case 0 :
		    	rx_get_128(timeArray);
		    	sec_time_1588[0] = transto1588(timeArray);//获取t1
		    	LogInfo(&sec_time_1588[0]);
		    	state = 1;//
			break;
		case 1://这是发送完delay_req产生的中断
			state = 2;
	    	tx_get_128(timeArray);
	    	sec_time_1588[2] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[2]);
			//证明delay_req发送完成,读取发送的时间t3
			break;
		case 2:
			//接收DelayResq
	    	rx_get_128(timeArray);
	    	sec_time_1588[3] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[3]);
	    	state = -1;
			break;



	}
	printf("zhongduan = %d \n",state);


#endif


    //////////////////////////////////////////////////////
	//完成中断，再将中断使能
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);

}


/*
 * crossbar_interrupt.c
 *
 *  Created on: 2015-11-13
 *      Author: Xaxdus
 */

#include"xil_io.h"
#include"xstatus.h"
#include "crossbar_interrupt.h"
#include "crossbar_1588.h"

static INTC Intc;//全局的INTC
extern Delay_Resp delay_resp;
extern u32 interruptflag;
extern  int state;
extern Time_1588 sec_time_1588[4];
/**
 * 	初始化中断配置
 */
int init_1588_Interrupt(){
	int result;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Config* IntcConfig;
	/* Initializethe interrupt controller driver so that it is ready to
	* use.
	* */
	IntcConfig =XScuGic_LookupConfig(INTC_DEVICE_ID);
	if (IntcConfig== NULL)
	{
		return XST_FAILURE;
	}
	/* Initializethe SCU and GIC to enable the desired interrupt
	* configuration.*/
	result =XScuGic_CfgInitialize(IntcInstancePtr, IntcConfig,
	IntcConfig->CpuBaseAddress);
	if (result !=XST_SUCCESS)
	{
		return XST_FAILURE;
	}
	XScuGic_SetPriorityTriggerType(IntcInstancePtr,crossbar_1588_interruptID,
	0xA0, 0x3);
	/* Connect the interrupt handler that will be called when an
	* interrupt occurs for the device. */
	result =XScuGic_Connect(IntcInstancePtr, crossbar_1588_interruptID,
	(Xil_ExceptionHandler)deal_1588_interrupt, 0);
	if (result !=XST_SUCCESS)
	{
		return result;
	}
	/* Enable the interrupt for the 1588 controller device. */
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);
	/* Initializethe exception table and register the interrupt controller
	* handler withthe exception table. */
	Xil_ExceptionInit();
	Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,
	(Xil_ExceptionHandler)INTC_HANDLER,IntcInstancePtr);
	/* Enablenon-critical exceptions */
	Xil_ExceptionEnable();
	return XST_SUCCESS;
}
/**
 *
 * 1588中断的处理函数
 */
void deal_1588_interrupt(void*InstancePtr){
	//进入中断之后先将中断关闭
	u32 timeArray[4];
	Time_1588 time_1588;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Disable(IntcInstancePtr,crossbar_1588_interruptID);
	///////////////////////////////////////////////////
	//来中断的时候,需要读取接收寄存器的数据
#if 0//作为主时钟的时候,主要是解析Delay_Req
	switch(state){
		case 0 :
				state = 1;//开始发送FollowUp
			break;
		case 1://这是发送完FollowUP产生的中断
			state = 2;//证明Followup发送完成
			break;
		case 2:
			//接收Delayreq
	    	rx_get_128(timeArray);
	    	time_1588 = transto1588(timeArray);
	    	pack1588_to_Delay_Resp(&time_1588,&delay_resp);
	    	state = 3;
			break;
		case 3:break;


	}



#else //次时钟
	switch(state){

		case -1:
				rx_get_128(timeArray);
				sec_time_1588[1] = transto1588(timeArray);//获取t2
				LogInfo(&sec_time_1588[1]);
				state = 0;

				break;
		case 0 :
		    	rx_get_128(timeArray);
		    	sec_time_1588[0] = transto1588(timeArray);//获取t1
		    	LogInfo(&sec_time_1588[0]);
		    	state = 1;//
			break;
		case 1://这是发送完delay_req产生的中断
			state = 2;
	    	tx_get_128(timeArray);
	    	sec_time_1588[2] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[2]);
			//证明delay_req发送完成,读取发送的时间t3
			break;
		case 2:
			//接收DelayResq
	    	rx_get_128(timeArray);
	    	sec_time_1588[3] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[3]);
	    	state = -1;
			break;



	}
	printf("zhongduan = %d \n",state);


#endif


    //////////////////////////////////////////////////////
	//完成中断，再将中断使能
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);

}


/*
 * crossbar_interrupt.c
 *
 *  Created on: 2015-11-13
 *      Author: Xaxdus
 */

#include"xil_io.h"
#include"xstatus.h"
#include "crossbar_interrupt.h"
#include "crossbar_1588.h"

static INTC Intc;//全局的INTC
extern Delay_Resp delay_resp;
extern u32 interruptflag;
extern  int state;
extern Time_1588 sec_time_1588[4];
/**
 * 	初始化中断配置
 */
int init_1588_Interrupt(){
	int result;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Config* IntcConfig;
	/* Initializethe interrupt controller driver so that it is ready to
	* use.
	* */
	IntcConfig =XScuGic_LookupConfig(INTC_DEVICE_ID);
	if (IntcConfig== NULL)
	{
		return XST_FAILURE;
	}
	/* Initializethe SCU and GIC to enable the desired interrupt
	* configuration.*/
	result =XScuGic_CfgInitialize(IntcInstancePtr, IntcConfig,
	IntcConfig->CpuBaseAddress);
	if (result !=XST_SUCCESS)
	{
		return XST_FAILURE;
	}
	XScuGic_SetPriorityTriggerType(IntcInstancePtr,crossbar_1588_interruptID,
	0xA0, 0x3);
	/* Connect the interrupt handler that will be called when an
	* interrupt occurs for the device. */
	result =XScuGic_Connect(IntcInstancePtr, crossbar_1588_interruptID,
	(Xil_ExceptionHandler)deal_1588_interrupt, 0);
	if (result !=XST_SUCCESS)
	{
		return result;
	}
	/* Enable the interrupt for the 1588 controller device. */
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);
	/* Initializethe exception table and register the interrupt controller
	* handler withthe exception table. */
	Xil_ExceptionInit();
	Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,
	(Xil_ExceptionHandler)INTC_HANDLER,IntcInstancePtr);
	/* Enablenon-critical exceptions */
	Xil_ExceptionEnable();
	return XST_SUCCESS;
}
/**
 *
 * 1588中断的处理函数
 */
void deal_1588_interrupt(void*InstancePtr){
	//进入中断之后先将中断关闭
	u32 timeArray[4];
	Time_1588 time_1588;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Disable(IntcInstancePtr,crossbar_1588_interruptID);
	///////////////////////////////////////////////////
	//来中断的时候,需要读取接收寄存器的数据
#if 0//作为主时钟的时候,主要是解析Delay_Req
	switch(state){
		case 0 :
				state = 1;//开始发送FollowUp
			break;
		case 1://这是发送完FollowUP产生的中断
			state = 2;//证明Followup发送完成
			break;
		case 2:
			//接收Delayreq
	    	rx_get_128(timeArray);
	    	time_1588 = transto1588(timeArray);
	    	pack1588_to_Delay_Resp(&time_1588,&delay_resp);
	    	state = 3;
			break;
		case 3:break;


	}



#else //次时钟
	switch(state){

		case -1:
				rx_get_128(timeArray);
				sec_time_1588[1] = transto1588(timeArray);//获取t2
				LogInfo(&sec_time_1588[1]);
				state = 0;

				break;
		case 0 :
		    	rx_get_128(timeArray);
		    	sec_time_1588[0] = transto1588(timeArray);//获取t1
		    	LogInfo(&sec_time_1588[0]);
		    	state = 1;//
			break;
		case 1://这是发送完delay_req产生的中断
			state = 2;
	    	tx_get_128(timeArray);
	    	sec_time_1588[2] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[2]);
			//证明delay_req发送完成,读取发送的时间t3
			break;
		case 2:
			//接收DelayResq
	    	rx_get_128(timeArray);
	    	sec_time_1588[3] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[3]);
	    	state = -1;
			break;



	}
	printf("zhongduan = %d \n",state);


#endif


    //////////////////////////////////////////////////////
	//完成中断，再将中断使能
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);

}


/*
 * crossbar_interrupt.c
 *
 *  Created on: 2015-11-13
 *      Author: Xaxdus
 */

#include"xil_io.h"
#include"xstatus.h"
#include "crossbar_interrupt.h"
#include "crossbar_1588.h"

static INTC Intc;//全局的INTC
extern Delay_Resp delay_resp;
extern u32 interruptflag;
extern  int state;
extern Time_1588 sec_time_1588[4];
/**
 * 	初始化中断配置
 */
int init_1588_Interrupt(){
	int result;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Config* IntcConfig;
	/* Initializethe interrupt controller driver so that it is ready to
	* use.
	* */
	IntcConfig =XScuGic_LookupConfig(INTC_DEVICE_ID);
	if (IntcConfig== NULL)
	{
		return XST_FAILURE;
	}
	/* Initializethe SCU and GIC to enable the desired interrupt
	* configuration.*/
	result =XScuGic_CfgInitialize(IntcInstancePtr, IntcConfig,
	IntcConfig->CpuBaseAddress);
	if (result !=XST_SUCCESS)
	{
		return XST_FAILURE;
	}
	XScuGic_SetPriorityTriggerType(IntcInstancePtr,crossbar_1588_interruptID,
	0xA0, 0x3);
	/* Connect the interrupt handler that will be called when an
	* interrupt occurs for the device. */
	result =XScuGic_Connect(IntcInstancePtr, crossbar_1588_interruptID,
	(Xil_ExceptionHandler)deal_1588_interrupt, 0);
	if (result !=XST_SUCCESS)
	{
		return result;
	}
	/* Enable the interrupt for the 1588 controller device. */
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);
	/* Initializethe exception table and register the interrupt controller
	* handler withthe exception table. */
	Xil_ExceptionInit();
	Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,
	(Xil_ExceptionHandler)INTC_HANDLER,IntcInstancePtr);
	/* Enablenon-critical exceptions */
	Xil_ExceptionEnable();
	return XST_SUCCESS;
}
/**
 *
 * 1588中断的处理函数
 */
void deal_1588_interrupt(void*InstancePtr){
	//进入中断之后先将中断关闭
	u32 timeArray[4];
	Time_1588 time_1588;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Disable(IntcInstancePtr,crossbar_1588_interruptID);
	///////////////////////////////////////////////////
	//来中断的时候,需要读取接收寄存器的数据
#if 0//作为主时钟的时候,主要是解析Delay_Req
	switch(state){
		case 0 :
				state = 1;//开始发送FollowUp
			break;
		case 1://这是发送完FollowUP产生的中断
			state = 2;//证明Followup发送完成
			break;
		case 2:
			//接收Delayreq
	    	rx_get_128(timeArray);
	    	time_1588 = transto1588(timeArray);
	    	pack1588_to_Delay_Resp(&time_1588,&delay_resp);
	    	state = 3;
			break;
		case 3:break;


	}



#else //次时钟
	switch(state){

		case -1:
				rx_get_128(timeArray);
				sec_time_1588[1] = transto1588(timeArray);//获取t2
				LogInfo(&sec_time_1588[1]);
				state = 0;

				break;
		case 0 :
		    	rx_get_128(timeArray);
		    	sec_time_1588[0] = transto1588(timeArray);//获取t1
		    	LogInfo(&sec_time_1588[0]);
		    	state = 1;//
			break;
		case 1://这是发送完delay_req产生的中断
			state = 2;
	    	tx_get_128(timeArray);
	    	sec_time_1588[2] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[2]);
			//证明delay_req发送完成,读取发送的时间t3
			break;
		case 2:
			//接收DelayResq
	    	rx_get_128(timeArray);
	    	sec_time_1588[3] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[3]);
	    	state = -1;
			break;



	}
	printf("zhongduan = %d \n",state);


#endif


    //////////////////////////////////////////////////////
	//完成中断，再将中断使能
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);

}


/*
 * crossbar_interrupt.c
 *
 *  Created on: 2015-11-13
 *      Author: Xaxdus
 */

#include"xil_io.h"
#include"xstatus.h"
#include "crossbar_interrupt.h"
#include "crossbar_1588.h"

static INTC Intc;//全局的INTC
extern Delay_Resp delay_resp;
extern u32 interruptflag;
extern  int state;
extern Time_1588 sec_time_1588[4];
/**
 * 	初始化中断配置
 */
int init_1588_Interrupt(){
	int result;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Config* IntcConfig;
	/* Initializethe interrupt controller driver so that it is ready to
	* use.
	* */
	IntcConfig =XScuGic_LookupConfig(INTC_DEVICE_ID);
	if (IntcConfig== NULL)
	{
		return XST_FAILURE;
	}
	/* Initializethe SCU and GIC to enable the desired interrupt
	* configuration.*/
	result =XScuGic_CfgInitialize(IntcInstancePtr, IntcConfig,
	IntcConfig->CpuBaseAddress);
	if (result !=XST_SUCCESS)
	{
		return XST_FAILURE;
	}
	XScuGic_SetPriorityTriggerType(IntcInstancePtr,crossbar_1588_interruptID,
	0xA0, 0x3);
	/* Connect the interrupt handler that will be called when an
	* interrupt occurs for the device. */
	result =XScuGic_Connect(IntcInstancePtr, crossbar_1588_interruptID,
	(Xil_ExceptionHandler)deal_1588_interrupt, 0);
	if (result !=XST_SUCCESS)
	{
		return result;
	}
	/* Enable the interrupt for the 1588 controller device. */
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);
	/* Initializethe exception table and register the interrupt controller
	* handler withthe exception table. */
	Xil_ExceptionInit();
	Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,
	(Xil_ExceptionHandler)INTC_HANDLER,IntcInstancePtr);
	/* Enablenon-critical exceptions */
	Xil_ExceptionEnable();
	return XST_SUCCESS;
}
/**
 *
 * 1588中断的处理函数
 */
void deal_1588_interrupt(void*InstancePtr){
	//进入中断之后先将中断关闭
	u32 timeArray[4];
	Time_1588 time_1588;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Disable(IntcInstancePtr,crossbar_1588_interruptID);
	///////////////////////////////////////////////////
	//来中断的时候,需要读取接收寄存器的数据
#if 0//作为主时钟的时候,主要是解析Delay_Req
	switch(state){
		case 0 :
				state = 1;//开始发送FollowUp
			break;
		case 1://这是发送完FollowUP产生的中断
			state = 2;//证明Followup发送完成
			break;
		case 2:
			//接收Delayreq
	    	rx_get_128(timeArray);
	    	time_1588 = transto1588(timeArray);
	    	pack1588_to_Delay_Resp(&time_1588,&delay_resp);
	    	state = 3;
			break;
		case 3:break;


	}



#else //次时钟
	switch(state){

		case -1:
				rx_get_128(timeArray);
				sec_time_1588[1] = transto1588(timeArray);//获取t2
				LogInfo(&sec_time_1588[1]);
				state = 0;

				break;
		case 0 :
		    	rx_get_128(timeArray);
		    	sec_time_1588[0] = transto1588(timeArray);//获取t1
		    	LogInfo(&sec_time_1588[0]);
		    	state = 1;//
			break;
		case 1://这是发送完delay_req产生的中断
			state = 2;
	    	tx_get_128(timeArray);
	    	sec_time_1588[2] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[2]);
			//证明delay_req发送完成,读取发送的时间t3
			break;
		case 2:
			//接收DelayResq
	    	rx_get_128(timeArray);
	    	sec_time_1588[3] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[3]);
	    	state = -1;
			break;



	}
	printf("zhongduan = %d \n",state);


#endif


    //////////////////////////////////////////////////////
	//完成中断，再将中断使能
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);

}


/*
 * crossbar_interrupt.c
 *
 *  Created on: 2015-11-13
 *      Author: Xaxdus
 */

#include"xil_io.h"
#include"xstatus.h"
#include "crossbar_interrupt.h"
#include "crossbar_1588.h"

static INTC Intc;//全局的INTC
extern Delay_Resp delay_resp;
extern u32 interruptflag;
extern  int state;
extern Time_1588 sec_time_1588[4];
/**
 * 	初始化中断配置
 */
int init_1588_Interrupt(){
	int result;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Config* IntcConfig;
	/* Initializethe interrupt controller driver so that it is ready to
	* use.
	* */
	IntcConfig =XScuGic_LookupConfig(INTC_DEVICE_ID);
	if (IntcConfig== NULL)
	{
		return XST_FAILURE;
	}
	/* Initializethe SCU and GIC to enable the desired interrupt
	* configuration.*/
	result =XScuGic_CfgInitialize(IntcInstancePtr, IntcConfig,
	IntcConfig->CpuBaseAddress);
	if (result !=XST_SUCCESS)
	{
		return XST_FAILURE;
	}
	XScuGic_SetPriorityTriggerType(IntcInstancePtr,crossbar_1588_interruptID,
	0xA0, 0x3);
	/* Connect the interrupt handler that will be called when an
	* interrupt occurs for the device. */
	result =XScuGic_Connect(IntcInstancePtr, crossbar_1588_interruptID,
	(Xil_ExceptionHandler)deal_1588_interrupt, 0);
	if (result !=XST_SUCCESS)
	{
		return result;
	}
	/* Enable the interrupt for the 1588 controller device. */
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);
	/* Initializethe exception table and register the interrupt controller
	* handler withthe exception table. */
	Xil_ExceptionInit();
	Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,
	(Xil_ExceptionHandler)INTC_HANDLER,IntcInstancePtr);
	/* Enablenon-critical exceptions */
	Xil_ExceptionEnable();
	return XST_SUCCESS;
}
/**
 *
 * 1588中断的处理函数
 */
void deal_1588_interrupt(void*InstancePtr){
	//进入中断之后先将中断关闭
	u32 timeArray[4];
	Time_1588 time_1588;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Disable(IntcInstancePtr,crossbar_1588_interruptID);
	///////////////////////////////////////////////////
	//来中断的时候,需要读取接收寄存器的数据
#if 0//作为主时钟的时候,主要是解析Delay_Req
	switch(state){
		case 0 :
				state = 1;//开始发送FollowUp
			break;
		case 1://这是发送完FollowUP产生的中断
			state = 2;//证明Followup发送完成
			break;
		case 2:
			//接收Delayreq
	    	rx_get_128(timeArray);
	    	time_1588 = transto1588(timeArray);
	    	pack1588_to_Delay_Resp(&time_1588,&delay_resp);
	    	state = 3;
			break;
		case 3:break;


	}



#else //次时钟
	switch(state){

		case -1:
				rx_get_128(timeArray);
				sec_time_1588[1] = transto1588(timeArray);//获取t2
				LogInfo(&sec_time_1588[1]);
				state = 0;

				break;
		case 0 :
		    	rx_get_128(timeArray);
		    	sec_time_1588[0] = transto1588(timeArray);//获取t1
		    	LogInfo(&sec_time_1588[0]);
		    	state = 1;//
			break;
		case 1://这是发送完delay_req产生的中断
			state = 2;
	    	tx_get_128(timeArray);
	    	sec_time_1588[2] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[2]);
			//证明delay_req发送完成,读取发送的时间t3
			break;
		case 2:
			//接收DelayResq
	    	rx_get_128(timeArray);
	    	sec_time_1588[3] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[3]);
	    	state = -1;
			break;



	}
	printf("zhongduan = %d \n",state);


#endif


    //////////////////////////////////////////////////////
	//完成中断，再将中断使能
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);

}


/*
 * crossbar_interrupt.c
 *
 *  Created on: 2015-11-13
 *      Author: Xaxdus
 */

#include"xil_io.h"
#include"xstatus.h"
#include "crossbar_interrupt.h"
#include "crossbar_1588.h"

static INTC Intc;//全局的INTC
extern Delay_Resp delay_resp;
extern u32 interruptflag;
extern  int state;
extern Time_1588 sec_time_1588[4];
/**
 * 	初始化中断配置
 */
int init_1588_Interrupt(){
	int result;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Config* IntcConfig;
	/* Initializethe interrupt controller driver so that it is ready to
	* use.
	* */
	IntcConfig =XScuGic_LookupConfig(INTC_DEVICE_ID);
	if (IntcConfig== NULL)
	{
		return XST_FAILURE;
	}
	/* Initializethe SCU and GIC to enable the desired interrupt
	* configuration.*/
	result =XScuGic_CfgInitialize(IntcInstancePtr, IntcConfig,
	IntcConfig->CpuBaseAddress);
	if (result !=XST_SUCCESS)
	{
		return XST_FAILURE;
	}
	XScuGic_SetPriorityTriggerType(IntcInstancePtr,crossbar_1588_interruptID,
	0xA0, 0x3);
	/* Connect the interrupt handler that will be called when an
	* interrupt occurs for the device. */
	result =XScuGic_Connect(IntcInstancePtr, crossbar_1588_interruptID,
	(Xil_ExceptionHandler)deal_1588_interrupt, 0);
	if (result !=XST_SUCCESS)
	{
		return result;
	}
	/* Enable the interrupt for the 1588 controller device. */
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);
	/* Initializethe exception table and register the interrupt controller
	* handler withthe exception table. */
	Xil_ExceptionInit();
	Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,
	(Xil_ExceptionHandler)INTC_HANDLER,IntcInstancePtr);
	/* Enablenon-critical exceptions */
	Xil_ExceptionEnable();
	return XST_SUCCESS;
}
/**
 *
 * 1588中断的处理函数
 */
void deal_1588_interrupt(void*InstancePtr){
	//进入中断之后先将中断关闭
	u32 timeArray[4];
	Time_1588 time_1588;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Disable(IntcInstancePtr,crossbar_1588_interruptID);
	///////////////////////////////////////////////////
	//来中断的时候,需要读取接收寄存器的数据
#if 0//作为主时钟的时候,主要是解析Delay_Req
	switch(state){
		case 0 :
				state = 1;//开始发送FollowUp
			break;
		case 1://这是发送完FollowUP产生的中断
			state = 2;//证明Followup发送完成
			break;
		case 2:
			//接收Delayreq
	    	rx_get_128(timeArray);
	    	time_1588 = transto1588(timeArray);
	    	pack1588_to_Delay_Resp(&time_1588,&delay_resp);
	    	state = 3;
			break;
		case 3:break;


	}



#else //次时钟
	switch(state){

		case -1:
				rx_get_128(timeArray);
				sec_time_1588[1] = transto1588(timeArray);//获取t2
				LogInfo(&sec_time_1588[1]);
				state = 0;

				break;
		case 0 :
		    	rx_get_128(timeArray);
		    	sec_time_1588[0] = transto1588(timeArray);//获取t1
		    	LogInfo(&sec_time_1588[0]);
		    	state = 1;//
			break;
		case 1://这是发送完delay_req产生的中断
			state = 2;
	    	tx_get_128(timeArray);
	    	sec_time_1588[2] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[2]);
			//证明delay_req发送完成,读取发送的时间t3
			break;
		case 2:
			//接收DelayResq
	    	rx_get_128(timeArray);
	    	sec_time_1588[3] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[3]);
	    	state = -1;
			break;



	}
	printf("zhongduan = %d \n",state);


#endif


    //////////////////////////////////////////////////////
	//完成中断，再将中断使能
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);

}


/*
 * crossbar_interrupt.c
 *
 *  Created on: 2015-11-13
 *      Author: Xaxdus
 */

#include"xil_io.h"
#include"xstatus.h"
#include "crossbar_interrupt.h"
#include "crossbar_1588.h"

static INTC Intc;//全局的INTC
extern Delay_Resp delay_resp;
extern u32 interruptflag;
extern  int state;
extern Time_1588 sec_time_1588[4];
/**
 * 	初始化中断配置
 */
int init_1588_Interrupt(){
	int result;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Config* IntcConfig;
	/* Initializethe interrupt controller driver so that it is ready to
	* use.
	* */
	IntcConfig =XScuGic_LookupConfig(INTC_DEVICE_ID);
	if (IntcConfig== NULL)
	{
		return XST_FAILURE;
	}
	/* Initializethe SCU and GIC to enable the desired interrupt
	* configuration.*/
	result =XScuGic_CfgInitialize(IntcInstancePtr, IntcConfig,
	IntcConfig->CpuBaseAddress);
	if (result !=XST_SUCCESS)
	{
		return XST_FAILURE;
	}
	XScuGic_SetPriorityTriggerType(IntcInstancePtr,crossbar_1588_interruptID,
	0xA0, 0x3);
	/* Connect the interrupt handler that will be called when an
	* interrupt occurs for the device. */
	result =XScuGic_Connect(IntcInstancePtr, crossbar_1588_interruptID,
	(Xil_ExceptionHandler)deal_1588_interrupt, 0);
	if (result !=XST_SUCCESS)
	{
		return result;
	}
	/* Enable the interrupt for the 1588 controller device. */
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);
	/* Initializethe exception table and register the interrupt controller
	* handler withthe exception table. */
	Xil_ExceptionInit();
	Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,
	(Xil_ExceptionHandler)INTC_HANDLER,IntcInstancePtr);
	/* Enablenon-critical exceptions */
	Xil_ExceptionEnable();
	return XST_SUCCESS;
}
/**
 *
 * 1588中断的处理函数
 */
void deal_1588_interrupt(void*InstancePtr){
	//进入中断之后先将中断关闭
	u32 timeArray[4];
	Time_1588 time_1588;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Disable(IntcInstancePtr,crossbar_1588_interruptID);
	///////////////////////////////////////////////////
	//来中断的时候,需要读取接收寄存器的数据
#if 0//作为主时钟的时候,主要是解析Delay_Req
	switch(state){
		case 0 :
				state = 1;//开始发送FollowUp
			break;
		case 1://这是发送完FollowUP产生的中断
			state = 2;//证明Followup发送完成
			break;
		case 2:
			//接收Delayreq
	    	rx_get_128(timeArray);
	    	time_1588 = transto1588(timeArray);
	    	pack1588_to_Delay_Resp(&time_1588,&delay_resp);
	    	state = 3;
			break;
		case 3:break;


	}



#else //次时钟
	switch(state){

		case -1:
				rx_get_128(timeArray);
				sec_time_1588[1] = transto1588(timeArray);//获取t2
				LogInfo(&sec_time_1588[1]);
				state = 0;

				break;
		case 0 :
		    	rx_get_128(timeArray);
		    	sec_time_1588[0] = transto1588(timeArray);//获取t1
		    	LogInfo(&sec_time_1588[0]);
		    	state = 1;//
			break;
		case 1://这是发送完delay_req产生的中断
			state = 2;
	    	tx_get_128(timeArray);
	    	sec_time_1588[2] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[2]);
			//证明delay_req发送完成,读取发送的时间t3
			break;
		case 2:
			//接收DelayResq
	    	rx_get_128(timeArray);
	    	sec_time_1588[3] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[3]);
	    	state = -1;
			break;



	}
	printf("zhongduan = %d \n",state);


#endif


    //////////////////////////////////////////////////////
	//完成中断，再将中断使能
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);

}


/*
 * crossbar_interrupt.c
 *
 *  Created on: 2015-11-13
 *      Author: Xaxdus
 */

#include"xil_io.h"
#include"xstatus.h"
#include "crossbar_interrupt.h"
#include "crossbar_1588.h"

static INTC Intc;//全局的INTC
extern Delay_Resp delay_resp;
extern u32 interruptflag;
extern  int state;
extern Time_1588 sec_time_1588[4];
/**
 * 	初始化中断配置
 */
int init_1588_Interrupt(){
	int result;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Config* IntcConfig;
	/* Initializethe interrupt controller driver so that it is ready to
	* use.
	* */
	IntcConfig =XScuGic_LookupConfig(INTC_DEVICE_ID);
	if (IntcConfig== NULL)
	{
		return XST_FAILURE;
	}
	/* Initializethe SCU and GIC to enable the desired interrupt
	* configuration.*/
	result =XScuGic_CfgInitialize(IntcInstancePtr, IntcConfig,
	IntcConfig->CpuBaseAddress);
	if (result !=XST_SUCCESS)
	{
		return XST_FAILURE;
	}
	XScuGic_SetPriorityTriggerType(IntcInstancePtr,crossbar_1588_interruptID,
	0xA0, 0x3);
	/* Connect the interrupt handler that will be called when an
	* interrupt occurs for the device. */
	result =XScuGic_Connect(IntcInstancePtr, crossbar_1588_interruptID,
	(Xil_ExceptionHandler)deal_1588_interrupt, 0);
	if (result !=XST_SUCCESS)
	{
		return result;
	}
	/* Enable the interrupt for the 1588 controller device. */
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);
	/* Initializethe exception table and register the interrupt controller
	* handler withthe exception table. */
	Xil_ExceptionInit();
	Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,
	(Xil_ExceptionHandler)INTC_HANDLER,IntcInstancePtr);
	/* Enablenon-critical exceptions */
	Xil_ExceptionEnable();
	return XST_SUCCESS;
}
/**
 *
 * 1588中断的处理函数
 */
void deal_1588_interrupt(void*InstancePtr){
	//进入中断之后先将中断关闭
	u32 timeArray[4];
	Time_1588 time_1588;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Disable(IntcInstancePtr,crossbar_1588_interruptID);
	///////////////////////////////////////////////////
	//来中断的时候,需要读取接收寄存器的数据
#if 0//作为主时钟的时候,主要是解析Delay_Req
	switch(state){
		case 0 :
				state = 1;//开始发送FollowUp
			break;
		case 1://这是发送完FollowUP产生的中断
			state = 2;//证明Followup发送完成
			break;
		case 2:
			//接收Delayreq
	    	rx_get_128(timeArray);
	    	time_1588 = transto1588(timeArray);
	    	pack1588_to_Delay_Resp(&time_1588,&delay_resp);
	    	state = 3;
			break;
		case 3:break;


	}



#else //次时钟
	switch(state){

		case -1:
				rx_get_128(timeArray);
				sec_time_1588[1] = transto1588(timeArray);//获取t2
				LogInfo(&sec_time_1588[1]);
				state = 0;

				break;
		case 0 :
		    	rx_get_128(timeArray);
		    	sec_time_1588[0] = transto1588(timeArray);//获取t1
		    	LogInfo(&sec_time_1588[0]);
		    	state = 1;//
			break;
		case 1://这是发送完delay_req产生的中断
			state = 2;
	    	tx_get_128(timeArray);
	    	sec_time_1588[2] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[2]);
			//证明delay_req发送完成,读取发送的时间t3
			break;
		case 2:
			//接收DelayResq
	    	rx_get_128(timeArray);
	    	sec_time_1588[3] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[3]);
	    	state = -1;
			break;



	}
	printf("zhongduan = %d \n",state);


#endif


    //////////////////////////////////////////////////////
	//完成中断，再将中断使能
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);

}


/*
 * crossbar_interrupt.c
 *
 *  Created on: 2015-11-13
 *      Author: Xaxdus
 */

#include"xil_io.h"
#include"xstatus.h"
#include "crossbar_interrupt.h"
#include "crossbar_1588.h"

static INTC Intc;//全局的INTC
extern Delay_Resp delay_resp;
extern u32 interruptflag;
extern  int state;
extern Time_1588 sec_time_1588[4];
/**
 * 	初始化中断配置
 */
int init_1588_Interrupt(){
	int result;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Config* IntcConfig;
	/* Initializethe interrupt controller driver so that it is ready to
	* use.
	* */
	IntcConfig =XScuGic_LookupConfig(INTC_DEVICE_ID);
	if (IntcConfig== NULL)
	{
		return XST_FAILURE;
	}
	/* Initializethe SCU and GIC to enable the desired interrupt
	* configuration.*/
	result =XScuGic_CfgInitialize(IntcInstancePtr, IntcConfig,
	IntcConfig->CpuBaseAddress);
	if (result !=XST_SUCCESS)
	{
		return XST_FAILURE;
	}
	XScuGic_SetPriorityTriggerType(IntcInstancePtr,crossbar_1588_interruptID,
	0xA0, 0x3);
	/* Connect the interrupt handler that will be called when an
	* interrupt occurs for the device. */
	result =XScuGic_Connect(IntcInstancePtr, crossbar_1588_interruptID,
	(Xil_ExceptionHandler)deal_1588_interrupt, 0);
	if (result !=XST_SUCCESS)
	{
		return result;
	}
	/* Enable the interrupt for the 1588 controller device. */
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);
	/* Initializethe exception table and register the interrupt controller
	* handler withthe exception table. */
	Xil_ExceptionInit();
	Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,
	(Xil_ExceptionHandler)INTC_HANDLER,IntcInstancePtr);
	/* Enablenon-critical exceptions */
	Xil_ExceptionEnable();
	return XST_SUCCESS;
}
/**
 *
 * 1588中断的处理函数
 */
void deal_1588_interrupt(void*InstancePtr){
	//进入中断之后先将中断关闭
	u32 timeArray[4];
	Time_1588 time_1588;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Disable(IntcInstancePtr,crossbar_1588_interruptID);
	///////////////////////////////////////////////////
	//来中断的时候,需要读取接收寄存器的数据
#if 0//作为主时钟的时候,主要是解析Delay_Req
	switch(state){
		case 0 :
				state = 1;//开始发送FollowUp
			break;
		case 1://这是发送完FollowUP产生的中断
			state = 2;//证明Followup发送完成
			break;
		case 2:
			//接收Delayreq
	    	rx_get_128(timeArray);
	    	time_1588 = transto1588(timeArray);
	    	pack1588_to_Delay_Resp(&time_1588,&delay_resp);
	    	state = 3;
			break;
		case 3:break;


	}



#else //次时钟
	switch(state){

		case -1:
				rx_get_128(timeArray);
				sec_time_1588[1] = transto1588(timeArray);//获取t2
				LogInfo(&sec_time_1588[1]);
				state = 0;

				break;
		case 0 :
		    	rx_get_128(timeArray);
		    	sec_time_1588[0] = transto1588(timeArray);//获取t1
		    	LogInfo(&sec_time_1588[0]);
		    	state = 1;//
			break;
		case 1://这是发送完delay_req产生的中断
			state = 2;
	    	tx_get_128(timeArray);
	    	sec_time_1588[2] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[2]);
			//证明delay_req发送完成,读取发送的时间t3
			break;
		case 2:
			//接收DelayResq
	    	rx_get_128(timeArray);
	    	sec_time_1588[3] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[3]);
	    	state = -1;
			break;



	}
	printf("zhongduan = %d \n",state);


#endif


    //////////////////////////////////////////////////////
	//完成中断，再将中断使能
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);

}


/*
 * crossbar_interrupt.c
 *
 *  Created on: 2015-11-13
 *      Author: Xaxdus
 */

#include"xil_io.h"
#include"xstatus.h"
#include "crossbar_interrupt.h"
#include "crossbar_1588.h"

static INTC Intc;//全局的INTC
extern Delay_Resp delay_resp;
extern u32 interruptflag;
extern  int state;
extern Time_1588 sec_time_1588[4];
/**
 * 	初始化中断配置
 */
int init_1588_Interrupt(){
	int result;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Config* IntcConfig;
	/* Initializethe interrupt controller driver so that it is ready to
	* use.
	* */
	IntcConfig =XScuGic_LookupConfig(INTC_DEVICE_ID);
	if (IntcConfig== NULL)
	{
		return XST_FAILURE;
	}
	/* Initializethe SCU and GIC to enable the desired interrupt
	* configuration.*/
	result =XScuGic_CfgInitialize(IntcInstancePtr, IntcConfig,
	IntcConfig->CpuBaseAddress);
	if (result !=XST_SUCCESS)
	{
		return XST_FAILURE;
	}
	XScuGic_SetPriorityTriggerType(IntcInstancePtr,crossbar_1588_interruptID,
	0xA0, 0x3);
	/* Connect the interrupt handler that will be called when an
	* interrupt occurs for the device. */
	result =XScuGic_Connect(IntcInstancePtr, crossbar_1588_interruptID,
	(Xil_ExceptionHandler)deal_1588_interrupt, 0);
	if (result !=XST_SUCCESS)
	{
		return result;
	}
	/* Enable the interrupt for the 1588 controller device. */
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);
	/* Initializethe exception table and register the interrupt controller
	* handler withthe exception table. */
	Xil_ExceptionInit();
	Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,
	(Xil_ExceptionHandler)INTC_HANDLER,IntcInstancePtr);
	/* Enablenon-critical exceptions */
	Xil_ExceptionEnable();
	return XST_SUCCESS;
}
/**
 *
 * 1588中断的处理函数
 */
void deal_1588_interrupt(void*InstancePtr){
	//进入中断之后先将中断关闭
	u32 timeArray[4];
	Time_1588 time_1588;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Disable(IntcInstancePtr,crossbar_1588_interruptID);
	///////////////////////////////////////////////////
	//来中断的时候,需要读取接收寄存器的数据
#if 0//作为主时钟的时候,主要是解析Delay_Req
	switch(state){
		case 0 :
				state = 1;//开始发送FollowUp
			break;
		case 1://这是发送完FollowUP产生的中断
			state = 2;//证明Followup发送完成
			break;
		case 2:
			//接收Delayreq
	    	rx_get_128(timeArray);
	    	time_1588 = transto1588(timeArray);
	    	pack1588_to_Delay_Resp(&time_1588,&delay_resp);
	    	state = 3;
			break;
		case 3:break;


	}



#else //次时钟
	switch(state){

		case -1:
				rx_get_128(timeArray);
				sec_time_1588[1] = transto1588(timeArray);//获取t2
				LogInfo(&sec_time_1588[1]);
				state = 0;

				break;
		case 0 :
		    	rx_get_128(timeArray);
		    	sec_time_1588[0] = transto1588(timeArray);//获取t1
		    	LogInfo(&sec_time_1588[0]);
		    	state = 1;//
			break;
		case 1://这是发送完delay_req产生的中断
			state = 2;
	    	tx_get_128(timeArray);
	    	sec_time_1588[2] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[2]);
			//证明delay_req发送完成,读取发送的时间t3
			break;
		case 2:
			//接收DelayResq
	    	rx_get_128(timeArray);
	    	sec_time_1588[3] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[3]);
	    	state = -1;
			break;



	}
	printf("zhongduan = %d \n",state);


#endif


    //////////////////////////////////////////////////////
	//完成中断，再将中断使能
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);

}


/*
 * crossbar_interrupt.c
 *
 *  Created on: 2015-11-13
 *      Author: Xaxdus
 */

#include"xil_io.h"
#include"xstatus.h"
#include "crossbar_interrupt.h"
#include "crossbar_1588.h"

static INTC Intc;//全局的INTC
extern Delay_Resp delay_resp;
extern u32 interruptflag;
extern  int state;
extern Time_1588 sec_time_1588[4];
/**
 * 	初始化中断配置
 */
int init_1588_Interrupt(){
	int result;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Config* IntcConfig;
	/* Initializethe interrupt controller driver so that it is ready to
	* use.
	* */
	IntcConfig =XScuGic_LookupConfig(INTC_DEVICE_ID);
	if (IntcConfig== NULL)
	{
		return XST_FAILURE;
	}
	/* Initializethe SCU and GIC to enable the desired interrupt
	* configuration.*/
	result =XScuGic_CfgInitialize(IntcInstancePtr, IntcConfig,
	IntcConfig->CpuBaseAddress);
	if (result !=XST_SUCCESS)
	{
		return XST_FAILURE;
	}
	XScuGic_SetPriorityTriggerType(IntcInstancePtr,crossbar_1588_interruptID,
	0xA0, 0x3);
	/* Connect the interrupt handler that will be called when an
	* interrupt occurs for the device. */
	result =XScuGic_Connect(IntcInstancePtr, crossbar_1588_interruptID,
	(Xil_ExceptionHandler)deal_1588_interrupt, 0);
	if (result !=XST_SUCCESS)
	{
		return result;
	}
	/* Enable the interrupt for the 1588 controller device. */
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);
	/* Initializethe exception table and register the interrupt controller
	* handler withthe exception table. */
	Xil_ExceptionInit();
	Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,
	(Xil_ExceptionHandler)INTC_HANDLER,IntcInstancePtr);
	/* Enablenon-critical exceptions */
	Xil_ExceptionEnable();
	return XST_SUCCESS;
}
/**
 *
 * 1588中断的处理函数
 */
void deal_1588_interrupt(void*InstancePtr){
	//进入中断之后先将中断关闭
	u32 timeArray[4];
	Time_1588 time_1588;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Disable(IntcInstancePtr,crossbar_1588_interruptID);
	///////////////////////////////////////////////////
	//来中断的时候,需要读取接收寄存器的数据
#if 0//作为主时钟的时候,主要是解析Delay_Req
	switch(state){
		case 0 :
				state = 1;//开始发送FollowUp
			break;
		case 1://这是发送完FollowUP产生的中断
			state = 2;//证明Followup发送完成
			break;
		case 2:
			//接收Delayreq
	    	rx_get_128(timeArray);
	    	time_1588 = transto1588(timeArray);
	    	pack1588_to_Delay_Resp(&time_1588,&delay_resp);
	    	state = 3;
			break;
		case 3:break;


	}



#else //次时钟
	switch(state){

		case -1:
				rx_get_128(timeArray);
				sec_time_1588[1] = transto1588(timeArray);//获取t2
				LogInfo(&sec_time_1588[1]);
				state = 0;

				break;
		case 0 :
		    	rx_get_128(timeArray);
		    	sec_time_1588[0] = transto1588(timeArray);//获取t1
		    	LogInfo(&sec_time_1588[0]);
		    	state = 1;//
			break;
		case 1://这是发送完delay_req产生的中断
			state = 2;
	    	tx_get_128(timeArray);
	    	sec_time_1588[2] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[2]);
			//证明delay_req发送完成,读取发送的时间t3
			break;
		case 2:
			//接收DelayResq
	    	rx_get_128(timeArray);
	    	sec_time_1588[3] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[3]);
	    	state = -1;
			break;



	}
	printf("zhongduan = %d \n",state);


#endif


    //////////////////////////////////////////////////////
	//完成中断，再将中断使能
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);

}


/*
 * crossbar_interrupt.c
 *
 *  Created on: 2015-11-13
 *      Author: Xaxdus
 */

#include"xil_io.h"
#include"xstatus.h"
#include "crossbar_interrupt.h"
#include "crossbar_1588.h"

static INTC Intc;//全局的INTC
extern Delay_Resp delay_resp;
extern u32 interruptflag;
extern  int state;
extern Time_1588 sec_time_1588[4];
/**
 * 	初始化中断配置
 */
int init_1588_Interrupt(){
	int result;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Config* IntcConfig;
	/* Initializethe interrupt controller driver so that it is ready to
	* use.
	* */
	IntcConfig =XScuGic_LookupConfig(INTC_DEVICE_ID);
	if (IntcConfig== NULL)
	{
		return XST_FAILURE;
	}
	/* Initializethe SCU and GIC to enable the desired interrupt
	* configuration.*/
	result =XScuGic_CfgInitialize(IntcInstancePtr, IntcConfig,
	IntcConfig->CpuBaseAddress);
	if (result !=XST_SUCCESS)
	{
		return XST_FAILURE;
	}
	XScuGic_SetPriorityTriggerType(IntcInstancePtr,crossbar_1588_interruptID,
	0xA0, 0x3);
	/* Connect the interrupt handler that will be called when an
	* interrupt occurs for the device. */
	result =XScuGic_Connect(IntcInstancePtr, crossbar_1588_interruptID,
	(Xil_ExceptionHandler)deal_1588_interrupt, 0);
	if (result !=XST_SUCCESS)
	{
		return result;
	}
	/* Enable the interrupt for the 1588 controller device. */
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);
	/* Initializethe exception table and register the interrupt controller
	* handler withthe exception table. */
	Xil_ExceptionInit();
	Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,
	(Xil_ExceptionHandler)INTC_HANDLER,IntcInstancePtr);
	/* Enablenon-critical exceptions */
	Xil_ExceptionEnable();
	return XST_SUCCESS;
}
/**
 *
 * 1588中断的处理函数
 */
void deal_1588_interrupt(void*InstancePtr){
	//进入中断之后先将中断关闭
	u32 timeArray[4];
	Time_1588 time_1588;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Disable(IntcInstancePtr,crossbar_1588_interruptID);
	///////////////////////////////////////////////////
	//来中断的时候,需要读取接收寄存器的数据
#if 0//作为主时钟的时候,主要是解析Delay_Req
	switch(state){
		case 0 :
				state = 1;//开始发送FollowUp
			break;
		case 1://这是发送完FollowUP产生的中断
			state = 2;//证明Followup发送完成
			break;
		case 2:
			//接收Delayreq
	    	rx_get_128(timeArray);
	    	time_1588 = transto1588(timeArray);
	    	pack1588_to_Delay_Resp(&time_1588,&delay_resp);
	    	state = 3;
			break;
		case 3:break;


	}



#else //次时钟
	switch(state){

		case -1:
				rx_get_128(timeArray);
				sec_time_1588[1] = transto1588(timeArray);//获取t2
				LogInfo(&sec_time_1588[1]);
				state = 0;

				break;
		case 0 :
		    	rx_get_128(timeArray);
		    	sec_time_1588[0] = transto1588(timeArray);//获取t1
		    	LogInfo(&sec_time_1588[0]);
		    	state = 1;//
			break;
		case 1://这是发送完delay_req产生的中断
			state = 2;
	    	tx_get_128(timeArray);
	    	sec_time_1588[2] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[2]);
			//证明delay_req发送完成,读取发送的时间t3
			break;
		case 2:
			//接收DelayResq
	    	rx_get_128(timeArray);
	    	sec_time_1588[3] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[3]);
	    	state = -1;
			break;



	}
	printf("zhongduan = %d \n",state);


#endif


    //////////////////////////////////////////////////////
	//完成中断，再将中断使能
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);

}


/*
 * crossbar_interrupt.c
 *
 *  Created on: 2015-11-13
 *      Author: Xaxdus
 */

#include"xil_io.h"
#include"xstatus.h"
#include "crossbar_interrupt.h"
#include "crossbar_1588.h"

static INTC Intc;//全局的INTC
extern Delay_Resp delay_resp;
extern u32 interruptflag;
extern  int state;
extern Time_1588 sec_time_1588[4];
/**
 * 	初始化中断配置
 */
int init_1588_Interrupt(){
	int result;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Config* IntcConfig;
	/* Initializethe interrupt controller driver so that it is ready to
	* use.
	* */
	IntcConfig =XScuGic_LookupConfig(INTC_DEVICE_ID);
	if (IntcConfig== NULL)
	{
		return XST_FAILURE;
	}
	/* Initializethe SCU and GIC to enable the desired interrupt
	* configuration.*/
	result =XScuGic_CfgInitialize(IntcInstancePtr, IntcConfig,
	IntcConfig->CpuBaseAddress);
	if (result !=XST_SUCCESS)
	{
		return XST_FAILURE;
	}
	XScuGic_SetPriorityTriggerType(IntcInstancePtr,crossbar_1588_interruptID,
	0xA0, 0x3);
	/* Connect the interrupt handler that will be called when an
	* interrupt occurs for the device. */
	result =XScuGic_Connect(IntcInstancePtr, crossbar_1588_interruptID,
	(Xil_ExceptionHandler)deal_1588_interrupt, 0);
	if (result !=XST_SUCCESS)
	{
		return result;
	}
	/* Enable the interrupt for the 1588 controller device. */
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);
	/* Initializethe exception table and register the interrupt controller
	* handler withthe exception table. */
	Xil_ExceptionInit();
	Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,
	(Xil_ExceptionHandler)INTC_HANDLER,IntcInstancePtr);
	/* Enablenon-critical exceptions */
	Xil_ExceptionEnable();
	return XST_SUCCESS;
}
/**
 *
 * 1588中断的处理函数
 */
void deal_1588_interrupt(void*InstancePtr){
	//进入中断之后先将中断关闭
	u32 timeArray[4];
	Time_1588 time_1588;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Disable(IntcInstancePtr,crossbar_1588_interruptID);
	///////////////////////////////////////////////////
	//来中断的时候,需要读取接收寄存器的数据
#if 0//作为主时钟的时候,主要是解析Delay_Req
	switch(state){
		case 0 :
				state = 1;//开始发送FollowUp
			break;
		case 1://这是发送完FollowUP产生的中断
			state = 2;//证明Followup发送完成
			break;
		case 2:
			//接收Delayreq
	    	rx_get_128(timeArray);
	    	time_1588 = transto1588(timeArray);
	    	pack1588_to_Delay_Resp(&time_1588,&delay_resp);
	    	state = 3;
			break;
		case 3:break;


	}



#else //次时钟
	switch(state){

		case -1:
				rx_get_128(timeArray);
				sec_time_1588[1] = transto1588(timeArray);//获取t2
				LogInfo(&sec_time_1588[1]);
				state = 0;

				break;
		case 0 :
		    	rx_get_128(timeArray);
		    	sec_time_1588[0] = transto1588(timeArray);//获取t1
		    	LogInfo(&sec_time_1588[0]);
		    	state = 1;//
			break;
		case 1://这是发送完delay_req产生的中断
			state = 2;
	    	tx_get_128(timeArray);
	    	sec_time_1588[2] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[2]);
			//证明delay_req发送完成,读取发送的时间t3
			break;
		case 2:
			//接收DelayResq
	    	rx_get_128(timeArray);
	    	sec_time_1588[3] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[3]);
	    	state = -1;
			break;



	}
	printf("zhongduan = %d \n",state);


#endif


    //////////////////////////////////////////////////////
	//完成中断，再将中断使能
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);

}


/*
 * crossbar_interrupt.c
 *
 *  Created on: 2015-11-13
 *      Author: Xaxdus
 */

#include"xil_io.h"
#include"xstatus.h"
#include "crossbar_interrupt.h"
#include "crossbar_1588.h"

static INTC Intc;//全局的INTC
extern Delay_Resp delay_resp;
extern u32 interruptflag;
extern  int state;
extern Time_1588 sec_time_1588[4];
/**
 * 	初始化中断配置
 */
int init_1588_Interrupt(){
	int result;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Config* IntcConfig;
	/* Initializethe interrupt controller driver so that it is ready to
	* use.
	* */
	IntcConfig =XScuGic_LookupConfig(INTC_DEVICE_ID);
	if (IntcConfig== NULL)
	{
		return XST_FAILURE;
	}
	/* Initializethe SCU and GIC to enable the desired interrupt
	* configuration.*/
	result =XScuGic_CfgInitialize(IntcInstancePtr, IntcConfig,
	IntcConfig->CpuBaseAddress);
	if (result !=XST_SUCCESS)
	{
		return XST_FAILURE;
	}
	XScuGic_SetPriorityTriggerType(IntcInstancePtr,crossbar_1588_interruptID,
	0xA0, 0x3);
	/* Connect the interrupt handler that will be called when an
	* interrupt occurs for the device. */
	result =XScuGic_Connect(IntcInstancePtr, crossbar_1588_interruptID,
	(Xil_ExceptionHandler)deal_1588_interrupt, 0);
	if (result !=XST_SUCCESS)
	{
		return result;
	}
	/* Enable the interrupt for the 1588 controller device. */
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);
	/* Initializethe exception table and register the interrupt controller
	* handler withthe exception table. */
	Xil_ExceptionInit();
	Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,
	(Xil_ExceptionHandler)INTC_HANDLER,IntcInstancePtr);
	/* Enablenon-critical exceptions */
	Xil_ExceptionEnable();
	return XST_SUCCESS;
}
/**
 *
 * 1588中断的处理函数
 */
void deal_1588_interrupt(void*InstancePtr){
	//进入中断之后先将中断关闭
	u32 timeArray[4];
	Time_1588 time_1588;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Disable(IntcInstancePtr,crossbar_1588_interruptID);
	///////////////////////////////////////////////////
	//来中断的时候,需要读取接收寄存器的数据
#if 0//作为主时钟的时候,主要是解析Delay_Req
	switch(state){
		case 0 :
				state = 1;//开始发送FollowUp
			break;
		case 1://这是发送完FollowUP产生的中断
			state = 2;//证明Followup发送完成
			break;
		case 2:
			//接收Delayreq
	    	rx_get_128(timeArray);
	    	time_1588 = transto1588(timeArray);
	    	pack1588_to_Delay_Resp(&time_1588,&delay_resp);
	    	state = 3;
			break;
		case 3:break;


	}



#else //次时钟
	switch(state){

		case -1:
				rx_get_128(timeArray);
				sec_time_1588[1] = transto1588(timeArray);//获取t2
				LogInfo(&sec_time_1588[1]);
				state = 0;

				break;
		case 0 :
		    	rx_get_128(timeArray);
		    	sec_time_1588[0] = transto1588(timeArray);//获取t1
		    	LogInfo(&sec_time_1588[0]);
		    	state = 1;//
			break;
		case 1://这是发送完delay_req产生的中断
			state = 2;
	    	tx_get_128(timeArray);
	    	sec_time_1588[2] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[2]);
			//证明delay_req发送完成,读取发送的时间t3
			break;
		case 2:
			//接收DelayResq
	    	rx_get_128(timeArray);
	    	sec_time_1588[3] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[3]);
	    	state = -1;
			break;



	}
	printf("zhongduan = %d \n",state);


#endif


    //////////////////////////////////////////////////////
	//完成中断，再将中断使能
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);

}


/*
 * crossbar_interrupt.c
 *
 *  Created on: 2015-11-13
 *      Author: Xaxdus
 */

#include"xil_io.h"
#include"xstatus.h"
#include "crossbar_interrupt.h"
#include "crossbar_1588.h"

static INTC Intc;//全局的INTC
extern Delay_Resp delay_resp;
extern u32 interruptflag;
extern  int state;
extern Time_1588 sec_time_1588[4];
/**
 * 	初始化中断配置
 */
int init_1588_Interrupt(){
	int result;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Config* IntcConfig;
	/* Initializethe interrupt controller driver so that it is ready to
	* use.
	* */
	IntcConfig =XScuGic_LookupConfig(INTC_DEVICE_ID);
	if (IntcConfig== NULL)
	{
		return XST_FAILURE;
	}
	/* Initializethe SCU and GIC to enable the desired interrupt
	* configuration.*/
	result =XScuGic_CfgInitialize(IntcInstancePtr, IntcConfig,
	IntcConfig->CpuBaseAddress);
	if (result !=XST_SUCCESS)
	{
		return XST_FAILURE;
	}
	XScuGic_SetPriorityTriggerType(IntcInstancePtr,crossbar_1588_interruptID,
	0xA0, 0x3);
	/* Connect the interrupt handler that will be called when an
	* interrupt occurs for the device. */
	result =XScuGic_Connect(IntcInstancePtr, crossbar_1588_interruptID,
	(Xil_ExceptionHandler)deal_1588_interrupt, 0);
	if (result !=XST_SUCCESS)
	{
		return result;
	}
	/* Enable the interrupt for the 1588 controller device. */
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);
	/* Initializethe exception table and register the interrupt controller
	* handler withthe exception table. */
	Xil_ExceptionInit();
	Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,
	(Xil_ExceptionHandler)INTC_HANDLER,IntcInstancePtr);
	/* Enablenon-critical exceptions */
	Xil_ExceptionEnable();
	return XST_SUCCESS;
}
/**
 *
 * 1588中断的处理函数
 */
void deal_1588_interrupt(void*InstancePtr){
	//进入中断之后先将中断关闭
	u32 timeArray[4];
	Time_1588 time_1588;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Disable(IntcInstancePtr,crossbar_1588_interruptID);
	///////////////////////////////////////////////////
	//来中断的时候,需要读取接收寄存器的数据
#if 0//作为主时钟的时候,主要是解析Delay_Req
	switch(state){
		case 0 :
				state = 1;//开始发送FollowUp
			break;
		case 1://这是发送完FollowUP产生的中断
			state = 2;//证明Followup发送完成
			break;
		case 2:
			//接收Delayreq
	    	rx_get_128(timeArray);
	    	time_1588 = transto1588(timeArray);
	    	pack1588_to_Delay_Resp(&time_1588,&delay_resp);
	    	state = 3;
			break;
		case 3:break;


	}



#else //次时钟
	switch(state){

		case -1:
				rx_get_128(timeArray);
				sec_time_1588[1] = transto1588(timeArray);//获取t2
				LogInfo(&sec_time_1588[1]);
				state = 0;

				break;
		case 0 :
		    	rx_get_128(timeArray);
		    	sec_time_1588[0] = transto1588(timeArray);//获取t1
		    	LogInfo(&sec_time_1588[0]);
		    	state = 1;//
			break;
		case 1://这是发送完delay_req产生的中断
			state = 2;
	    	tx_get_128(timeArray);
	    	sec_time_1588[2] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[2]);
			//证明delay_req发送完成,读取发送的时间t3
			break;
		case 2:
			//接收DelayResq
	    	rx_get_128(timeArray);
	    	sec_time_1588[3] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[3]);
	    	state = -1;
			break;



	}
	printf("zhongduan = %d \n",state);


#endif


    //////////////////////////////////////////////////////
	//完成中断，再将中断使能
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);

}


/*
 * crossbar_interrupt.c
 *
 *  Created on: 2015-11-13
 *      Author: Xaxdus
 */

#include"xil_io.h"
#include"xstatus.h"
#include "crossbar_interrupt.h"
#include "crossbar_1588.h"

static INTC Intc;//全局的INTC
extern Delay_Resp delay_resp;
extern u32 interruptflag;
extern  int state;
extern Time_1588 sec_time_1588[4];
/**
 * 	初始化中断配置
 */
int init_1588_Interrupt(){
	int result;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Config* IntcConfig;
	/* Initializethe interrupt controller driver so that it is ready to
	* use.
	* */
	IntcConfig =XScuGic_LookupConfig(INTC_DEVICE_ID);
	if (IntcConfig== NULL)
	{
		return XST_FAILURE;
	}
	/* Initializethe SCU and GIC to enable the desired interrupt
	* configuration.*/
	result =XScuGic_CfgInitialize(IntcInstancePtr, IntcConfig,
	IntcConfig->CpuBaseAddress);
	if (result !=XST_SUCCESS)
	{
		return XST_FAILURE;
	}
	XScuGic_SetPriorityTriggerType(IntcInstancePtr,crossbar_1588_interruptID,
	0xA0, 0x3);
	/* Connect the interrupt handler that will be called when an
	* interrupt occurs for the device. */
	result =XScuGic_Connect(IntcInstancePtr, crossbar_1588_interruptID,
	(Xil_ExceptionHandler)deal_1588_interrupt, 0);
	if (result !=XST_SUCCESS)
	{
		return result;
	}
	/* Enable the interrupt for the 1588 controller device. */
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);
	/* Initializethe exception table and register the interrupt controller
	* handler withthe exception table. */
	Xil_ExceptionInit();
	Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,
	(Xil_ExceptionHandler)INTC_HANDLER,IntcInstancePtr);
	/* Enablenon-critical exceptions */
	Xil_ExceptionEnable();
	return XST_SUCCESS;
}
/**
 *
 * 1588中断的处理函数
 */
void deal_1588_interrupt(void*InstancePtr){
	//进入中断之后先将中断关闭
	u32 timeArray[4];
	Time_1588 time_1588;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Disable(IntcInstancePtr,crossbar_1588_interruptID);
	///////////////////////////////////////////////////
	//来中断的时候,需要读取接收寄存器的数据
#if 0//作为主时钟的时候,主要是解析Delay_Req
	switch(state){
		case 0 :
				state = 1;//开始发送FollowUp
			break;
		case 1://这是发送完FollowUP产生的中断
			state = 2;//证明Followup发送完成
			break;
		case 2:
			//接收Delayreq
	    	rx_get_128(timeArray);
	    	time_1588 = transto1588(timeArray);
	    	pack1588_to_Delay_Resp(&time_1588,&delay_resp);
	    	state = 3;
			break;
		case 3:break;


	}



#else //次时钟
	switch(state){

		case -1:
				rx_get_128(timeArray);
				sec_time_1588[1] = transto1588(timeArray);//获取t2
				LogInfo(&sec_time_1588[1]);
				state = 0;

				break;
		case 0 :
		    	rx_get_128(timeArray);
		    	sec_time_1588[0] = transto1588(timeArray);//获取t1
		    	LogInfo(&sec_time_1588[0]);
		    	state = 1;//
			break;
		case 1://这是发送完delay_req产生的中断
			state = 2;
	    	tx_get_128(timeArray);
	    	sec_time_1588[2] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[2]);
			//证明delay_req发送完成,读取发送的时间t3
			break;
		case 2:
			//接收DelayResq
	    	rx_get_128(timeArray);
	    	sec_time_1588[3] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[3]);
	    	state = -1;
			break;



	}
	printf("zhongduan = %d \n",state);


#endif


    //////////////////////////////////////////////////////
	//完成中断，再将中断使能
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);

}


/*
 * crossbar_interrupt.c
 *
 *  Created on: 2015-11-13
 *      Author: Xaxdus
 */

#include"xil_io.h"
#include"xstatus.h"
#include "crossbar_interrupt.h"
#include "crossbar_1588.h"

static INTC Intc;//全局的INTC
extern Delay_Resp delay_resp;
extern u32 interruptflag;
extern  int state;
extern Time_1588 sec_time_1588[4];
/**
 * 	初始化中断配置
 */
int init_1588_Interrupt(){
	int result;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Config* IntcConfig;
	/* Initializethe interrupt controller driver so that it is ready to
	* use.
	* */
	IntcConfig =XScuGic_LookupConfig(INTC_DEVICE_ID);
	if (IntcConfig== NULL)
	{
		return XST_FAILURE;
	}
	/* Initializethe SCU and GIC to enable the desired interrupt
	* configuration.*/
	result =XScuGic_CfgInitialize(IntcInstancePtr, IntcConfig,
	IntcConfig->CpuBaseAddress);
	if (result !=XST_SUCCESS)
	{
		return XST_FAILURE;
	}
	XScuGic_SetPriorityTriggerType(IntcInstancePtr,crossbar_1588_interruptID,
	0xA0, 0x3);
	/* Connect the interrupt handler that will be called when an
	* interrupt occurs for the device. */
	result =XScuGic_Connect(IntcInstancePtr, crossbar_1588_interruptID,
	(Xil_ExceptionHandler)deal_1588_interrupt, 0);
	if (result !=XST_SUCCESS)
	{
		return result;
	}
	/* Enable the interrupt for the 1588 controller device. */
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);
	/* Initializethe exception table and register the interrupt controller
	* handler withthe exception table. */
	Xil_ExceptionInit();
	Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,
	(Xil_ExceptionHandler)INTC_HANDLER,IntcInstancePtr);
	/* Enablenon-critical exceptions */
	Xil_ExceptionEnable();
	return XST_SUCCESS;
}
/**
 *
 * 1588中断的处理函数
 */
void deal_1588_interrupt(void*InstancePtr){
	//进入中断之后先将中断关闭
	u32 timeArray[4];
	Time_1588 time_1588;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Disable(IntcInstancePtr,crossbar_1588_interruptID);
	///////////////////////////////////////////////////
	//来中断的时候,需要读取接收寄存器的数据
#if 0//作为主时钟的时候,主要是解析Delay_Req
	switch(state){
		case 0 :
				state = 1;//开始发送FollowUp
			break;
		case 1://这是发送完FollowUP产生的中断
			state = 2;//证明Followup发送完成
			break;
		case 2:
			//接收Delayreq
	    	rx_get_128(timeArray);
	    	time_1588 = transto1588(timeArray);
	    	pack1588_to_Delay_Resp(&time_1588,&delay_resp);
	    	state = 3;
			break;
		case 3:break;


	}



#else //次时钟
	switch(state){

		case -1:
				rx_get_128(timeArray);
				sec_time_1588[1] = transto1588(timeArray);//获取t2
				LogInfo(&sec_time_1588[1]);
				state = 0;

				break;
		case 0 :
		    	rx_get_128(timeArray);
		    	sec_time_1588[0] = transto1588(timeArray);//获取t1
		    	LogInfo(&sec_time_1588[0]);
		    	state = 1;//
			break;
		case 1://这是发送完delay_req产生的中断
			state = 2;
	    	tx_get_128(timeArray);
	    	sec_time_1588[2] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[2]);
			//证明delay_req发送完成,读取发送的时间t3
			break;
		case 2:
			//接收DelayResq
	    	rx_get_128(timeArray);
	    	sec_time_1588[3] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[3]);
	    	state = -1;
			break;



	}
	printf("zhongduan = %d \n",state);


#endif


    //////////////////////////////////////////////////////
	//完成中断，再将中断使能
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);

}


/*
 * crossbar_interrupt.c
 *
 *  Created on: 2015-11-13
 *      Author: Xaxdus
 */

#include"xil_io.h"
#include"xstatus.h"
#include "crossbar_interrupt.h"
#include "crossbar_1588.h"

static INTC Intc;//全局的INTC
extern Delay_Resp delay_resp;
extern u32 interruptflag;
extern  int state;
extern Time_1588 sec_time_1588[4];
/**
 * 	初始化中断配置
 */
int init_1588_Interrupt(){
	int result;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Config* IntcConfig;
	/* Initializethe interrupt controller driver so that it is ready to
	* use.
	* */
	IntcConfig =XScuGic_LookupConfig(INTC_DEVICE_ID);
	if (IntcConfig== NULL)
	{
		return XST_FAILURE;
	}
	/* Initializethe SCU and GIC to enable the desired interrupt
	* configuration.*/
	result =XScuGic_CfgInitialize(IntcInstancePtr, IntcConfig,
	IntcConfig->CpuBaseAddress);
	if (result !=XST_SUCCESS)
	{
		return XST_FAILURE;
	}
	XScuGic_SetPriorityTriggerType(IntcInstancePtr,crossbar_1588_interruptID,
	0xA0, 0x3);
	/* Connect the interrupt handler that will be called when an
	* interrupt occurs for the device. */
	result =XScuGic_Connect(IntcInstancePtr, crossbar_1588_interruptID,
	(Xil_ExceptionHandler)deal_1588_interrupt, 0);
	if (result !=XST_SUCCESS)
	{
		return result;
	}
	/* Enable the interrupt for the 1588 controller device. */
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);
	/* Initializethe exception table and register the interrupt controller
	* handler withthe exception table. */
	Xil_ExceptionInit();
	Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,
	(Xil_ExceptionHandler)INTC_HANDLER,IntcInstancePtr);
	/* Enablenon-critical exceptions */
	Xil_ExceptionEnable();
	return XST_SUCCESS;
}
/**
 *
 * 1588中断的处理函数
 */
void deal_1588_interrupt(void*InstancePtr){
	//进入中断之后先将中断关闭
	u32 timeArray[4];
	Time_1588 time_1588;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Disable(IntcInstancePtr,crossbar_1588_interruptID);
	///////////////////////////////////////////////////
	//来中断的时候,需要读取接收寄存器的数据
#if 0//作为主时钟的时候,主要是解析Delay_Req
	switch(state){
		case 0 :
				state = 1;//开始发送FollowUp
			break;
		case 1://这是发送完FollowUP产生的中断
			state = 2;//证明Followup发送完成
			break;
		case 2:
			//接收Delayreq
	    	rx_get_128(timeArray);
	    	time_1588 = transto1588(timeArray);
	    	pack1588_to_Delay_Resp(&time_1588,&delay_resp);
	    	state = 3;
			break;
		case 3:break;


	}



#else //次时钟
	switch(state){

		case -1:
				rx_get_128(timeArray);
				sec_time_1588[1] = transto1588(timeArray);//获取t2
				LogInfo(&sec_time_1588[1]);
				state = 0;

				break;
		case 0 :
		    	rx_get_128(timeArray);
		    	sec_time_1588[0] = transto1588(timeArray);//获取t1
		    	LogInfo(&sec_time_1588[0]);
		    	state = 1;//
			break;
		case 1://这是发送完delay_req产生的中断
			state = 2;
	    	tx_get_128(timeArray);
	    	sec_time_1588[2] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[2]);
			//证明delay_req发送完成,读取发送的时间t3
			break;
		case 2:
			//接收DelayResq
	    	rx_get_128(timeArray);
	    	sec_time_1588[3] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[3]);
	    	state = -1;
			break;



	}
	printf("zhongduan = %d \n",state);


#endif


    //////////////////////////////////////////////////////
	//完成中断，再将中断使能
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);

}


/*
 * crossbar_interrupt.c
 *
 *  Created on: 2015-11-13
 *      Author: Xaxdus
 */

#include"xil_io.h"
#include"xstatus.h"
#include "crossbar_interrupt.h"
#include "crossbar_1588.h"

static INTC Intc;//全局的INTC
extern Delay_Resp delay_resp;
extern u32 interruptflag;
extern  int state;
extern Time_1588 sec_time_1588[4];
/**
 * 	初始化中断配置
 */
int init_1588_Interrupt(){
	int result;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Config* IntcConfig;
	/* Initializethe interrupt controller driver so that it is ready to
	* use.
	* */
	IntcConfig =XScuGic_LookupConfig(INTC_DEVICE_ID);
	if (IntcConfig== NULL)
	{
		return XST_FAILURE;
	}
	/* Initializethe SCU and GIC to enable the desired interrupt
	* configuration.*/
	result =XScuGic_CfgInitialize(IntcInstancePtr, IntcConfig,
	IntcConfig->CpuBaseAddress);
	if (result !=XST_SUCCESS)
	{
		return XST_FAILURE;
	}
	XScuGic_SetPriorityTriggerType(IntcInstancePtr,crossbar_1588_interruptID,
	0xA0, 0x3);
	/* Connect the interrupt handler that will be called when an
	* interrupt occurs for the device. */
	result =XScuGic_Connect(IntcInstancePtr, crossbar_1588_interruptID,
	(Xil_ExceptionHandler)deal_1588_interrupt, 0);
	if (result !=XST_SUCCESS)
	{
		return result;
	}
	/* Enable the interrupt for the 1588 controller device. */
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);
	/* Initializethe exception table and register the interrupt controller
	* handler withthe exception table. */
	Xil_ExceptionInit();
	Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,
	(Xil_ExceptionHandler)INTC_HANDLER,IntcInstancePtr);
	/* Enablenon-critical exceptions */
	Xil_ExceptionEnable();
	return XST_SUCCESS;
}
/**
 *
 * 1588中断的处理函数
 */
void deal_1588_interrupt(void*InstancePtr){
	//进入中断之后先将中断关闭
	u32 timeArray[4];
	Time_1588 time_1588;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Disable(IntcInstancePtr,crossbar_1588_interruptID);
	///////////////////////////////////////////////////
	//来中断的时候,需要读取接收寄存器的数据
#if 0//作为主时钟的时候,主要是解析Delay_Req
	switch(state){
		case 0 :
				state = 1;//开始发送FollowUp
			break;
		case 1://这是发送完FollowUP产生的中断
			state = 2;//证明Followup发送完成
			break;
		case 2:
			//接收Delayreq
	    	rx_get_128(timeArray);
	    	time_1588 = transto1588(timeArray);
	    	pack1588_to_Delay_Resp(&time_1588,&delay_resp);
	    	state = 3;
			break;
		case 3:break;


	}



#else //次时钟
	switch(state){

		case -1:
				rx_get_128(timeArray);
				sec_time_1588[1] = transto1588(timeArray);//获取t2
				LogInfo(&sec_time_1588[1]);
				state = 0;

				break;
		case 0 :
		    	rx_get_128(timeArray);
		    	sec_time_1588[0] = transto1588(timeArray);//获取t1
		    	LogInfo(&sec_time_1588[0]);
		    	state = 1;//
			break;
		case 1://这是发送完delay_req产生的中断
			state = 2;
	    	tx_get_128(timeArray);
	    	sec_time_1588[2] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[2]);
			//证明delay_req发送完成,读取发送的时间t3
			break;
		case 2:
			//接收DelayResq
	    	rx_get_128(timeArray);
	    	sec_time_1588[3] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[3]);
	    	state = -1;
			break;



	}
	printf("zhongduan = %d \n",state);


#endif


    //////////////////////////////////////////////////////
	//完成中断，再将中断使能
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);

}


/*
 * crossbar_interrupt.c
 *
 *  Created on: 2015-11-13
 *      Author: Xaxdus
 */

#include"xil_io.h"
#include"xstatus.h"
#include "crossbar_interrupt.h"
#include "crossbar_1588.h"

static INTC Intc;//全局的INTC
extern Delay_Resp delay_resp;
extern u32 interruptflag;
extern  int state;
extern Time_1588 sec_time_1588[4];
/**
 * 	初始化中断配置
 */
int init_1588_Interrupt(){
	int result;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Config* IntcConfig;
	/* Initializethe interrupt controller driver so that it is ready to
	* use.
	* */
	IntcConfig =XScuGic_LookupConfig(INTC_DEVICE_ID);
	if (IntcConfig== NULL)
	{
		return XST_FAILURE;
	}
	/* Initializethe SCU and GIC to enable the desired interrupt
	* configuration.*/
	result =XScuGic_CfgInitialize(IntcInstancePtr, IntcConfig,
	IntcConfig->CpuBaseAddress);
	if (result !=XST_SUCCESS)
	{
		return XST_FAILURE;
	}
	XScuGic_SetPriorityTriggerType(IntcInstancePtr,crossbar_1588_interruptID,
	0xA0, 0x3);
	/* Connect the interrupt handler that will be called when an
	* interrupt occurs for the device. */
	result =XScuGic_Connect(IntcInstancePtr, crossbar_1588_interruptID,
	(Xil_ExceptionHandler)deal_1588_interrupt, 0);
	if (result !=XST_SUCCESS)
	{
		return result;
	}
	/* Enable the interrupt for the 1588 controller device. */
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);
	/* Initializethe exception table and register the interrupt controller
	* handler withthe exception table. */
	Xil_ExceptionInit();
	Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,
	(Xil_ExceptionHandler)INTC_HANDLER,IntcInstancePtr);
	/* Enablenon-critical exceptions */
	Xil_ExceptionEnable();
	return XST_SUCCESS;
}
/**
 *
 * 1588中断的处理函数
 */
void deal_1588_interrupt(void*InstancePtr){
	//进入中断之后先将中断关闭
	u32 timeArray[4];
	Time_1588 time_1588;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Disable(IntcInstancePtr,crossbar_1588_interruptID);
	///////////////////////////////////////////////////
	//来中断的时候,需要读取接收寄存器的数据
#if 0//作为主时钟的时候,主要是解析Delay_Req
	switch(state){
		case 0 :
				state = 1;//开始发送FollowUp
			break;
		case 1://这是发送完FollowUP产生的中断
			state = 2;//证明Followup发送完成
			break;
		case 2:
			//接收Delayreq
	    	rx_get_128(timeArray);
	    	time_1588 = transto1588(timeArray);
	    	pack1588_to_Delay_Resp(&time_1588,&delay_resp);
	    	state = 3;
			break;
		case 3:break;


	}



#else //次时钟
	switch(state){

		case -1:
				rx_get_128(timeArray);
				sec_time_1588[1] = transto1588(timeArray);//获取t2
				LogInfo(&sec_time_1588[1]);
				state = 0;

				break;
		case 0 :
		    	rx_get_128(timeArray);
		    	sec_time_1588[0] = transto1588(timeArray);//获取t1
		    	LogInfo(&sec_time_1588[0]);
		    	state = 1;//
			break;
		case 1://这是发送完delay_req产生的中断
			state = 2;
	    	tx_get_128(timeArray);
	    	sec_time_1588[2] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[2]);
			//证明delay_req发送完成,读取发送的时间t3
			break;
		case 2:
			//接收DelayResq
	    	rx_get_128(timeArray);
	    	sec_time_1588[3] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[3]);
	    	state = -1;
			break;



	}
	printf("zhongduan = %d \n",state);


#endif


    //////////////////////////////////////////////////////
	//完成中断，再将中断使能
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);

}


/*
 * crossbar_interrupt.c
 *
 *  Created on: 2015-11-13
 *      Author: Xaxdus
 */

#include"xil_io.h"
#include"xstatus.h"
#include "crossbar_interrupt.h"
#include "crossbar_1588.h"

static INTC Intc;//全局的INTC
extern Delay_Resp delay_resp;
extern u32 interruptflag;
extern  int state;
extern Time_1588 sec_time_1588[4];
/**
 * 	初始化中断配置
 */
int init_1588_Interrupt(){
	int result;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Config* IntcConfig;
	/* Initializethe interrupt controller driver so that it is ready to
	* use.
	* */
	IntcConfig =XScuGic_LookupConfig(INTC_DEVICE_ID);
	if (IntcConfig== NULL)
	{
		return XST_FAILURE;
	}
	/* Initializethe SCU and GIC to enable the desired interrupt
	* configuration.*/
	result =XScuGic_CfgInitialize(IntcInstancePtr, IntcConfig,
	IntcConfig->CpuBaseAddress);
	if (result !=XST_SUCCESS)
	{
		return XST_FAILURE;
	}
	XScuGic_SetPriorityTriggerType(IntcInstancePtr,crossbar_1588_interruptID,
	0xA0, 0x3);
	/* Connect the interrupt handler that will be called when an
	* interrupt occurs for the device. */
	result =XScuGic_Connect(IntcInstancePtr, crossbar_1588_interruptID,
	(Xil_ExceptionHandler)deal_1588_interrupt, 0);
	if (result !=XST_SUCCESS)
	{
		return result;
	}
	/* Enable the interrupt for the 1588 controller device. */
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);
	/* Initializethe exception table and register the interrupt controller
	* handler withthe exception table. */
	Xil_ExceptionInit();
	Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,
	(Xil_ExceptionHandler)INTC_HANDLER,IntcInstancePtr);
	/* Enablenon-critical exceptions */
	Xil_ExceptionEnable();
	return XST_SUCCESS;
}
/**
 *
 * 1588中断的处理函数
 */
void deal_1588_interrupt(void*InstancePtr){
	//进入中断之后先将中断关闭
	u32 timeArray[4];
	Time_1588 time_1588;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Disable(IntcInstancePtr,crossbar_1588_interruptID);
	///////////////////////////////////////////////////
	//来中断的时候,需要读取接收寄存器的数据
#if 0//作为主时钟的时候,主要是解析Delay_Req
	switch(state){
		case 0 :
				state = 1;//开始发送FollowUp
			break;
		case 1://这是发送完FollowUP产生的中断
			state = 2;//证明Followup发送完成
			break;
		case 2:
			//接收Delayreq
	    	rx_get_128(timeArray);
	    	time_1588 = transto1588(timeArray);
	    	pack1588_to_Delay_Resp(&time_1588,&delay_resp);
	    	state = 3;
			break;
		case 3:break;


	}



#else //次时钟
	switch(state){

		case -1:
				rx_get_128(timeArray);
				sec_time_1588[1] = transto1588(timeArray);//获取t2
				LogInfo(&sec_time_1588[1]);
				state = 0;

				break;
		case 0 :
		    	rx_get_128(timeArray);
		    	sec_time_1588[0] = transto1588(timeArray);//获取t1
		    	LogInfo(&sec_time_1588[0]);
		    	state = 1;//
			break;
		case 1://这是发送完delay_req产生的中断
			state = 2;
	    	tx_get_128(timeArray);
	    	sec_time_1588[2] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[2]);
			//证明delay_req发送完成,读取发送的时间t3
			break;
		case 2:
			//接收DelayResq
	    	rx_get_128(timeArray);
	    	sec_time_1588[3] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[3]);
	    	state = -1;
			break;



	}
	printf("zhongduan = %d \n",state);


#endif


    //////////////////////////////////////////////////////
	//完成中断，再将中断使能
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);

}


/*
 * crossbar_interrupt.c
 *
 *  Created on: 2015-11-13
 *      Author: Xaxdus
 */

#include"xil_io.h"
#include"xstatus.h"
#include "crossbar_interrupt.h"
#include "crossbar_1588.h"

static INTC Intc;//全局的INTC
extern Delay_Resp delay_resp;
extern u32 interruptflag;
extern  int state;
extern Time_1588 sec_time_1588[4];
/**
 * 	初始化中断配置
 */
int init_1588_Interrupt(){
	int result;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Config* IntcConfig;
	/* Initializethe interrupt controller driver so that it is ready to
	* use.
	* */
	IntcConfig =XScuGic_LookupConfig(INTC_DEVICE_ID);
	if (IntcConfig== NULL)
	{
		return XST_FAILURE;
	}
	/* Initializethe SCU and GIC to enable the desired interrupt
	* configuration.*/
	result =XScuGic_CfgInitialize(IntcInstancePtr, IntcConfig,
	IntcConfig->CpuBaseAddress);
	if (result !=XST_SUCCESS)
	{
		return XST_FAILURE;
	}
	XScuGic_SetPriorityTriggerType(IntcInstancePtr,crossbar_1588_interruptID,
	0xA0, 0x3);
	/* Connect the interrupt handler that will be called when an
	* interrupt occurs for the device. */
	result =XScuGic_Connect(IntcInstancePtr, crossbar_1588_interruptID,
	(Xil_ExceptionHandler)deal_1588_interrupt, 0);
	if (result !=XST_SUCCESS)
	{
		return result;
	}
	/* Enable the interrupt for the 1588 controller device. */
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);
	/* Initializethe exception table and register the interrupt controller
	* handler withthe exception table. */
	Xil_ExceptionInit();
	Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,
	(Xil_ExceptionHandler)INTC_HANDLER,IntcInstancePtr);
	/* Enablenon-critical exceptions */
	Xil_ExceptionEnable();
	return XST_SUCCESS;
}
/**
 *
 * 1588中断的处理函数
 */
void deal_1588_interrupt(void*InstancePtr){
	//进入中断之后先将中断关闭
	u32 timeArray[4];
	Time_1588 time_1588;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Disable(IntcInstancePtr,crossbar_1588_interruptID);
	///////////////////////////////////////////////////
	//来中断的时候,需要读取接收寄存器的数据
#if 0//作为主时钟的时候,主要是解析Delay_Req
	switch(state){
		case 0 :
				state = 1;//开始发送FollowUp
			break;
		case 1://这是发送完FollowUP产生的中断
			state = 2;//证明Followup发送完成
			break;
		case 2:
			//接收Delayreq
	    	rx_get_128(timeArray);
	    	time_1588 = transto1588(timeArray);
	    	pack1588_to_Delay_Resp(&time_1588,&delay_resp);
	    	state = 3;
			break;
		case 3:break;


	}



#else //次时钟
	switch(state){

		case -1:
				rx_get_128(timeArray);
				sec_time_1588[1] = transto1588(timeArray);//获取t2
				LogInfo(&sec_time_1588[1]);
				state = 0;

				break;
		case 0 :
		    	rx_get_128(timeArray);
		    	sec_time_1588[0] = transto1588(timeArray);//获取t1
		    	LogInfo(&sec_time_1588[0]);
		    	state = 1;//
			break;
		case 1://这是发送完delay_req产生的中断
			state = 2;
	    	tx_get_128(timeArray);
	    	sec_time_1588[2] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[2]);
			//证明delay_req发送完成,读取发送的时间t3
			break;
		case 2:
			//接收DelayResq
	    	rx_get_128(timeArray);
	    	sec_time_1588[3] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[3]);
	    	state = -1;
			break;



	}
	printf("zhongduan = %d \n",state);


#endif


    //////////////////////////////////////////////////////
	//完成中断，再将中断使能
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);

}

/*
 * crossbar_interrupt.c
 *
 *  Created on: 2015-11-13
 *      Author: Xaxdus
 */

#include"xil_io.h"
#include"xstatus.h"
#include "crossbar_interrupt.h"
#include "crossbar_1588.h"

static INTC Intc;//全局的INTC
extern Delay_Resp delay_resp;
extern u32 interruptflag;
extern  int state;
extern Time_1588 sec_time_1588[4];
/**
 * 	初始化中断配置
 */
int init_1588_Interrupt(){
	int result;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Config* IntcConfig;
	/* Initializethe interrupt controller driver so that it is ready to
	* use.
	* */
	IntcConfig =XScuGic_LookupConfig(INTC_DEVICE_ID);
	if (IntcConfig== NULL)
	{
		return XST_FAILURE;
	}
	/* Initializethe SCU and GIC to enable the desired interrupt
	* configuration.*/
	result =XScuGic_CfgInitialize(IntcInstancePtr, IntcConfig,
	IntcConfig->CpuBaseAddress);
	if (result !=XST_SUCCESS)
	{
		return XST_FAILURE;
	}
	XScuGic_SetPriorityTriggerType(IntcInstancePtr,crossbar_1588_interruptID,
	0xA0, 0x3);
	/* Connect the interrupt handler that will be called when an
	* interrupt occurs for the device. */
	result =XScuGic_Connect(IntcInstancePtr, crossbar_1588_interruptID,
	(Xil_ExceptionHandler)deal_1588_interrupt, 0);
	if (result !=XST_SUCCESS)
	{
		return result;
	}
	/* Enable the interrupt for the 1588 controller device. */
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);
	/* Initializethe exception table and register the interrupt controller
	* handler withthe exception table. */
	Xil_ExceptionInit();
	Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,
	(Xil_ExceptionHandler)INTC_HANDLER,IntcInstancePtr);
	/* Enablenon-critical exceptions */
	Xil_ExceptionEnable();
	return XST_SUCCESS;
}
/**
 *
 * 1588中断的处理函数
 */
void deal_1588_interrupt(void*InstancePtr){
	//进入中断之后先将中断关闭
	u32 timeArray[4];
	Time_1588 time_1588;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Disable(IntcInstancePtr,crossbar_1588_interruptID);
	///////////////////////////////////////////////////
	//来中断的时候,需要读取接收寄存器的数据
#if 0//作为主时钟的时候,主要是解析Delay_Req
	switch(state){
		case 0 :
				state = 1;//开始发送FollowUp
			break;
		case 1://这是发送完FollowUP产生的中断
			state = 2;//证明Followup发送完成
			break;
		case 2:
			//接收Delayreq
	    	rx_get_128(timeArray);
	    	time_1588 = transto1588(timeArray);
	    	pack1588_to_Delay_Resp(&time_1588,&delay_resp);
	    	state = 3;
			break;
		case 3:break;


	}



#else //次时钟
	switch(state){

		case -1:
				rx_get_128(timeArray);
				sec_time_1588[1] = transto1588(timeArray);//获取t2
				LogInfo(&sec_time_1588[1]);
				state = 0;

				break;
		case 0 :
		    	rx_get_128(timeArray);
		    	sec_time_1588[0] = transto1588(timeArray);//获取t1
		    	LogInfo(&sec_time_1588[0]);
		    	state = 1;//
			break;
		case 1://这是发送完delay_req产生的中断
			state = 2;
	    	tx_get_128(timeArray);
	    	sec_time_1588[2] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[2]);
			//证明delay_req发送完成,读取发送的时间t3
			break;
		case 2:
			//接收DelayResq
	    	rx_get_128(timeArray);
	    	sec_time_1588[3] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[3]);
	    	state = -1;
			break;



	}
	printf("zhongduan = %d \n",state);


#endif


    //////////////////////////////////////////////////////
	//完成中断，再将中断使能
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);

}


/*
 * crossbar_interrupt.c
 *
 *  Created on: 2015-11-13
 *      Author: Xaxdus
 */

#include"xil_io.h"
#include"xstatus.h"
#include "crossbar_interrupt.h"
#include "crossbar_1588.h"

static INTC Intc;//全局的INTC
extern Delay_Resp delay_resp;
extern u32 interruptflag;
extern  int state;
extern Time_1588 sec_time_1588[4];
/**
 * 	初始化中断配置
 */
int init_1588_Interrupt(){
	int result;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Config* IntcConfig;
	/* Initializethe interrupt controller driver so that it is ready to
	* use.
	* */
	IntcConfig =XScuGic_LookupConfig(INTC_DEVICE_ID);
	if (IntcConfig== NULL)
	{
		return XST_FAILURE;
	}
	/* Initializethe SCU and GIC to enable the desired interrupt
	* configuration.*/
	result =XScuGic_CfgInitialize(IntcInstancePtr, IntcConfig,
	IntcConfig->CpuBaseAddress);
	if (result !=XST_SUCCESS)
	{
		return XST_FAILURE;
	}
	XScuGic_SetPriorityTriggerType(IntcInstancePtr,crossbar_1588_interruptID,
	0xA0, 0x3);
	/* Connect the interrupt handler that will be called when an
	* interrupt occurs for the device. */
	result =XScuGic_Connect(IntcInstancePtr, crossbar_1588_interruptID,
	(Xil_ExceptionHandler)deal_1588_interrupt, 0);
	if (result !=XST_SUCCESS)
	{
		return result;
	}
	/* Enable the interrupt for the 1588 controller device. */
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);
	/* Initializethe exception table and register the interrupt controller
	* handler withthe exception table. */
	Xil_ExceptionInit();
	Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,
	(Xil_ExceptionHandler)INTC_HANDLER,IntcInstancePtr);
	/* Enablenon-critical exceptions */
	Xil_ExceptionEnable();
	return XST_SUCCESS;
}
/**
 *
 * 1588中断的处理函数
 */
void deal_1588_interrupt(void*InstancePtr){
	//进入中断之后先将中断关闭
	u32 timeArray[4];
	Time_1588 time_1588;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Disable(IntcInstancePtr,crossbar_1588_interruptID);
	///////////////////////////////////////////////////
	//来中断的时候,需要读取接收寄存器的数据
#if 0//作为主时钟的时候,主要是解析Delay_Req
	switch(state){
		case 0 :
				state = 1;//开始发送FollowUp
			break;
		case 1://这是发送完FollowUP产生的中断
			state = 2;//证明Followup发送完成
			break;
		case 2:
			//接收Delayreq
	    	rx_get_128(timeArray);
	    	time_1588 = transto1588(timeArray);
	    	pack1588_to_Delay_Resp(&time_1588,&delay_resp);
	    	state = 3;
			break;
		case 3:break;


	}



#else //次时钟
	switch(state){

		case -1:
				rx_get_128(timeArray);
				sec_time_1588[1] = transto1588(timeArray);//获取t2
				LogInfo(&sec_time_1588[1]);
				state = 0;

				break;
		case 0 :
		    	rx_get_128(timeArray);
		    	sec_time_1588[0] = transto1588(timeArray);//获取t1
		    	LogInfo(&sec_time_1588[0]);
		    	state = 1;//
			break;
		case 1://这是发送完delay_req产生的中断
			state = 2;
	    	tx_get_128(timeArray);
	    	sec_time_1588[2] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[2]);
			//证明delay_req发送完成,读取发送的时间t3
			break;
		case 2:
			//接收DelayResq
	    	rx_get_128(timeArray);
	    	sec_time_1588[3] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[3]);
	    	state = -1;
			break;



	}
	printf("zhongduan = %d \n",state);


#endif


    //////////////////////////////////////////////////////
	//完成中断，再将中断使能
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);

}



/*
 * crossbar_interrupt.c
 *
 *  Created on: 2015-11-13
 *      Author: Xaxdus
 */

#include"xil_io.h"
#include"xstatus.h"
#include "crossbar_interrupt.h"
#include "crossbar_1588.h"

static INTC Intc;//全局的INTC
extern Delay_Resp delay_resp;
extern u32 interruptflag;
extern  int state;
extern Time_1588 sec_time_1588[4];
/**
 * 	初始化中断配置
 */
int init_1588_Interrupt(){
	int result;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Config* IntcConfig;
	/* Initializethe interrupt controller driver so that it is ready to
	* use.
	* */
	IntcConfig =XScuGic_LookupConfig(INTC_DEVICE_ID);
	if (IntcConfig== NULL)
	{
		return XST_FAILURE;
	}
	/* Initializethe SCU and GIC to enable the desired interrupt
	* configuration.*/
	result =XScuGic_CfgInitialize(IntcInstancePtr, IntcConfig,
	IntcConfig->CpuBaseAddress);
	if (result !=XST_SUCCESS)
	{
		return XST_FAILURE;
	}
	XScuGic_SetPriorityTriggerType(IntcInstancePtr,crossbar_1588_interruptID,
	0xA0, 0x3);
	/* Connect the interrupt handler that will be called when an
	* interrupt occurs for the device. */
	result =XScuGic_Connect(IntcInstancePtr, crossbar_1588_interruptID,
	(Xil_ExceptionHandler)deal_1588_interrupt, 0);
	if (result !=XST_SUCCESS)
	{
		return result;
	}
	/* Enable the interrupt for the 1588 controller device. */
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);
	/* Initializethe exception table and register the interrupt controller
	* handler withthe exception table. */
	Xil_ExceptionInit();
	Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,
	(Xil_ExceptionHandler)INTC_HANDLER,IntcInstancePtr);
	/* Enablenon-critical exceptions */
	Xil_ExceptionEnable();
	return XST_SUCCESS;
}
/**
 *
 * 1588中断的处理函数
 */
void deal_1588_interrupt(void*InstancePtr){
	//进入中断之后先将中断关闭
	u32 timeArray[4];
	Time_1588 time_1588;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Disable(IntcInstancePtr,crossbar_1588_interruptID);
	///////////////////////////////////////////////////
	//来中断的时候,需要读取接收寄存器的数据
#if 0//作为主时钟的时候,主要是解析Delay_Req
	switch(state){
		case 0 :
				state = 1;//开始发送FollowUp
			break;
		case 1://这是发送完FollowUP产生的中断
			state = 2;//证明Followup发送完成
			break;
		case 2:
			//接收Delayreq
	    	rx_get_128(timeArray);
	    	time_1588 = transto1588(timeArray);
	    	pack1588_to_Delay_Resp(&time_1588,&delay_resp);
	    	state = 3;
			break;
		case 3:break;


	}



#else //次时钟
	switch(state){

		case -1:
				rx_get_128(timeArray);
				sec_time_1588[1] = transto1588(timeArray);//获取t2
				LogInfo(&sec_time_1588[1]);
				state = 0;

				break;
		case 0 :
		    	rx_get_128(timeArray);
		    	sec_time_1588[0] = transto1588(timeArray);//获取t1
		    	LogInfo(&sec_time_1588[0]);
		    	state = 1;//
			break;
		case 1://这是发送完delay_req产生的中断
			state = 2;
	    	tx_get_128(timeArray);
	    	sec_time_1588[2] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[2]);
			//证明delay_req发送完成,读取发送的时间t3
			break;
		case 2:
			//接收DelayResq
	    	rx_get_128(timeArray);
	    	sec_time_1588[3] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[3]);
	    	state = -1;
			break;



	}
	printf("zhongduan = %d \n",state);


#endif


    //////////////////////////////////////////////////////
	//完成中断，再将中断使能
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);

}


/*
 * crossbar_interrupt.c
 *
 *  Created on: 2015-11-13
 *      Author: Xaxdus
 */

#include"xil_io.h"
#include"xstatus.h"
#include "crossbar_interrupt.h"
#include "crossbar_1588.h"

static INTC Intc;//全局的INTC
extern Delay_Resp delay_resp;
extern u32 interruptflag;
extern  int state;
extern Time_1588 sec_time_1588[4];
/**
 * 	初始化中断配置
 */
int init_1588_Interrupt(){
	int result;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Config* IntcConfig;
	/* Initializethe interrupt controller driver so that it is ready to
	* use.
	* */
	IntcConfig =XScuGic_LookupConfig(INTC_DEVICE_ID);
	if (IntcConfig== NULL)
	{
		return XST_FAILURE;
	}
	/* Initializethe SCU and GIC to enable the desired interrupt
	* configuration.*/
	result =XScuGic_CfgInitialize(IntcInstancePtr, IntcConfig,
	IntcConfig->CpuBaseAddress);
	if (result !=XST_SUCCESS)
	{
		return XST_FAILURE;
	}
	XScuGic_SetPriorityTriggerType(IntcInstancePtr,crossbar_1588_interruptID,
	0xA0, 0x3);
	/* Connect the interrupt handler that will be called when an
	* interrupt occurs for the device. */
	result =XScuGic_Connect(IntcInstancePtr, crossbar_1588_interruptID,
	(Xil_ExceptionHandler)deal_1588_interrupt, 0);
	if (result !=XST_SUCCESS)
	{
		return result;
	}
	/* Enable the interrupt for the 1588 controller device. */
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);
	/* Initializethe exception table and register the interrupt controller
	* handler withthe exception table. */
	Xil_ExceptionInit();
	Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,
	(Xil_ExceptionHandler)INTC_HANDLER,IntcInstancePtr);
	/* Enablenon-critical exceptions */
	Xil_ExceptionEnable();
	return XST_SUCCESS;
}
/**
 *
 * 1588中断的处理函数
 */
void deal_1588_interrupt(void*InstancePtr){
	//进入中断之后先将中断关闭
	u32 timeArray[4];
	Time_1588 time_1588;
	INTC* IntcInstancePtr = &Intc;
	XScuGic_Disable(IntcInstancePtr,crossbar_1588_interruptID);
	///////////////////////////////////////////////////
	//来中断的时候,需要读取接收寄存器的数据
#if 0//作为主时钟的时候,主要是解析Delay_Req
	switch(state){
		case 0 :
				state = 1;//开始发送FollowUp
			break;
		case 1://这是发送完FollowUP产生的中断
			state = 2;//证明Followup发送完成
			break;
		case 2:
			//接收Delayreq
	    	rx_get_128(timeArray);
	    	time_1588 = transto1588(timeArray);
	    	pack1588_to_Delay_Resp(&time_1588,&delay_resp);
	    	state = 3;
			break;
		case 3:break;


	}



#else //次时钟
	switch(state){

		case -1:
				rx_get_128(timeArray);
				sec_time_1588[1] = transto1588(timeArray);//获取t2
				LogInfo(&sec_time_1588[1]);
				state = 0;

				break;
		case 0 :
		    	rx_get_128(timeArray);
		    	sec_time_1588[0] = transto1588(timeArray);//获取t1
		    	LogInfo(&sec_time_1588[0]);
		    	state = 1;//
			break;
		case 1://这是发送完delay_req产生的中断
			state = 2;
	    	tx_get_128(timeArray);
	    	sec_time_1588[2] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[2]);
			//证明delay_req发送完成,读取发送的时间t3
			break;
		case 2:
			//接收DelayResq
	    	rx_get_128(timeArray);
	    	sec_time_1588[3] = transto1588(timeArray);
	    	LogInfo(&sec_time_1588[3]);
	    	state = -1;
			break;



	}
	printf("zhongduan = %d \n",state);


#endif


    //////////////////////////////////////////////////////
	//完成中断，再将中断使能
	XScuGic_Enable(IntcInstancePtr,crossbar_1588_interruptID);

}


