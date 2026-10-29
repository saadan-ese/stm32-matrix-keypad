/**
 ******************************************************************************
 * @file    main.c
 * @brief   4x4 matrix keypad interfacing with STM32F407 (register level)
 *          Rows    R1-R4 -> PD0-PD3  (output)
 *          Columns C1-C4 -> PD8-PD11 (input, internal pull-up)
 *          Pressed key is printed over SWV ITM Data Console (Port 0)
 ******************************************************************************
 */

#include <stdint.h>
#include <stdio.h>

/* Simple software debounce delay: ~150 ms @ 16 MHz internal RC oscillator
 * (7 instructions ~ 0.5 us per iteration -> 2000 iterations/ms -> 150 ms = 300000) */
void delay(void)
{
	for (volatile uint32_t i = 0; i < 300000; i++);
}

int main(void)
{
	/* Peripheral register addresses */
	uint32_t volatile *const pGPIODModeReg  = (uint32_t*)(0x40020C00);          /* GPIOD_MODER   */
	uint32_t volatile *const pInPutDataReg  = (uint32_t*)(0x40020C00 + 0x10);   /* GPIOD_IDR     */
	uint32_t volatile *const pOutPutDataReg = (uint32_t*)(0x40020C00 + 0x14);   /* GPIOD_ODR     */
	uint32_t volatile *const pClockCtrlReg  = (uint32_t*)(0x40023800 + 0x30);   /* RCC_AHB1ENR   */
	uint32_t volatile *const pPullupDownReg = (uint32_t*)(0x40020C00 + 0x0C);   /* GPIOD_PUPDR   */

	/* Key map: keymap[row][column] */
	const char keymap[4][4] = {
		{'1', '2', '3', 'A'},
		{'4', '5', '6', 'B'},
		{'7', '8', '9', 'C'},
		{'*', '0', '#', 'D'}
	};

	/* 1. Enable the peripheral clock of GPIOD peripheral */
	*pClockCtrlReg |= (1 << 3);

	/* 2. Configure PD0, PD1, PD2, PD3 as output (rows) */
	*pGPIODModeReg &= ~(0xFF);      /* clear */
	*pGPIODModeReg |= 0x55;         /* set   */

	/* 3. Configure PD8, PD9, PD10, PD11 as input (columns) */
	*pGPIODModeReg &= ~(0xFF << 16);

	/* 4. Enable internal pull-up resistors for PD8, PD9, PD10, PD11 */
	*pPullupDownReg &= ~(0xFF << 16);
	*pPullupDownReg |= (0x55 << 16);

	while (1)
	{
		for (uint8_t row = 0; row < 4; row++)
		{
			/* Make all rows HIGH */
			*pOutPutDataReg |= 0x0F;

			/* Make the current row LOW (R1=PD0, R2=PD1, R3=PD2, R4=PD3) */
			*pOutPutDataReg &= ~(1 << row);

			/* Scan the columns: C1=PD8, C2=PD9, C3=PD10, C4=PD11 */
			for (uint8_t col = 0; col < 4; col++)
			{
				/* Column reads LOW -> key at (row, col) is pressed */
				if (!(*pInPutDataReg & (1 << (8 + col))))
				{
					delay();   /* debounce */
					printf("%c\n", keymap[row][col]);
				}
			}
		}
	}

	return 0;
}
