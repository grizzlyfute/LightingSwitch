#include <avr/io.h>
#include <avr/interrupt.h>
#include <avr/eeprom.h>
#include <avr/pgmspace.h>

// ---------- Defines
#define DEBUG_LED 0
#define MOVE_SENSOR 1
#define POTENTIOMETER 2
#define RELAY 3
#define PHOTORESITOR 4

// --------- structures
typedef struct EepromContents EepromContents;
struct EepromContents
{
	uint8_t time_hysteris; // in minutes
};

// ---------- global variables
EepromContents eepromContents EEMEM =
{
	.time_hysteris = 10,
};

// ---------- ISR
static volatile uint16_t main_counter = 0;
ISR(TIM0_COMPA_vect)
{
	main_counter++;
}

// -------- helpers
int8_t compareWithOverflow(uint16_t a, uint16_t b, uint16_t i)
{
	i = b + i;
	if (i >= b)
		return (a < b || a >= i);
	else
		return (a < b && a >= i);
}

uint16_t normalize0_1024(uint16_t x)
{
	if (x > 2048)
		x = 0;
	else if (x > 1024)
		x = 1023;
	return x;
}

uint16_t absDiff(uint16_t a, uint16_t b)
{
	if (a > b)
		return a - b;
	else
		return b - a;
}

// -------- functions
uint16_t getMainCounter(void)
{
	uint16_t counter;
	TIMSK &= ~(1 << OCIE0A); // Timer/Counter0 Output Compare Match A Interrupt Enable
	counter = main_counter;
	TIMSK |= (1 << OCIE0A);
	return counter;
}

// ADC 0 - 1024
uint16_t readAdcChannel(uint8_t pin)
{
	ADMUX &= ~((1 << MUX3) | (1 << MUX2) | (1 << MUX1) | (1 << MUX0));
	if (pin == POTENTIOMETER)
	{
		ADMUX |= (1 << MUX0); // Pin 2
	}
	else if (pin == PHOTORESITOR)
	{
		ADMUX |= (1 << MUX1); // Pin 4
	}
	else
	{
		// Gnd
		ADMUX |= (1 << MUX3) | (1 << MUX2) | (1 << MUX0);
	}

	// Start A2D Conversion single shot
	ADCSRA |= (1 << ADSC);
	// Wait for conversion to complete
	while (ADCSRA & (1 << ADSC));
	return ADC;
}

// output a number beetween 0 - 31 (5 bits) in base 2. long is 1, short is 0 (big endian)
void output(uint16_t currentTime, uint8_t n)
{
	static uint16_t delay = 0;
	static uint16_t recTime = 0;
	static int8_t seq = -1;
	if (compareWithOverflow(currentTime, recTime, delay))
	{
		recTime = currentTime;
		if (seq < 0)
		{
			seq = 10;
			PORTB &= ~(1 << DEBUG_LED);
			delay = 1000;
		}
		else if (seq % 2 == 1)
		{
			PORTB &= ~(1 << DEBUG_LED);
			delay = 300;
		}
		else if (n & (1 << (seq / 2)))
		{
			PORTB |= (1 << DEBUG_LED);
			delay = 1000;
		}
		else
		{
			PORTB |= (1 << DEBUG_LED);
			delay = 300;
		}
		seq--;
	}
}

int main(void)
{
	cli();
	// period = 1 / (freq / (prescal * OCRA)) = prescal * OCRA / freq
	// => OCRA = period / prescal * freq
	// Interrupt on counter0 every 1 ms with 1Mhz for frequency (default fuse)
	TCCR0A = (1 << WGM01); //Clear timer on compare match (CTC)
	TCCR0B = (1 << CS01); //use 8 as prescalar
	OCR0A = 125;
	TIMSK = (1 << OCIE0A); // Timer/Counter0 Output Compare Match A Interrupt Enable

	// Prepare Input / output
	DDRB = (1 << RELAY) | (1 << DEBUG_LED);
	// Internal pull up & shut off any effector
	PORTB = (1 << PHOTORESITOR) | (1 << MOVE_SENSOR) | (1 << POTENTIOMETER);

	// ADC ref Vcc, no left adjust
	ADMUX = 0;
	// Enable Analog digital converter - 16 prescalar for ADC clock (~62.5 kHz [50kHz-200kHz] with 1Mhz as main clock)
	ADCSRA = (1 << ADEN) | (1 << ADPS2);
	// Single ended input. Not free runnning mode (ADTATE = 0)
	ADCSRB = 0;
	DIDR0 = 0;

	sei();

	uint16_t time = 0;
	uint16_t tmp = 0xFFFF;
	uint16_t recordedTime = 0;
	// en minutes
	uint8_t absence_cnt = 0;
	uint8_t hysterisTime = 0;
	uint16_t last_potentiometer = 0;

	struct
	{
		uint8_t init:2;
		uint8_t analog:1;
		uint8_t present:1;
		uint8_t light:1;
		uint8_t hysterisChanged:1;
		uint8_t unused:2;
	} flags =
	{
		.init = 1,
		.analog = 0,
		.light = 0,
		.present = 1,
		.hysterisChanged = 0,
		.unused = 0,
	};

	while (1)
	{
		// timer
		time = getMainCounter();

		// analog in
		tmp = readAdcChannel(POTENTIOMETER);
		if (absDiff(tmp, last_potentiometer) > 20)
		{
			last_potentiometer = tmp;
			recordedTime = time;
			flags.analog = 1;
		}

		// init sequence
		if (flags.init == 1)
		{
			// load time hysteris from EEPROM
			hysterisTime = eeprom_read_byte(&eepromContents.time_hysteris);
			recordedTime = time;
			flags.init = 2;
			flags.analog = 0;

			// relay off
			PORTB &= ~(1 << RELAY);
		}
		else if (flags.init == 2)
		{
			if (flags.analog)
			{
				flags.analog = 0;
				if (hysterisTime != (20 * last_potentiometer / 1024))
				{
					flags.hysterisChanged = 1;
					hysterisTime = (20 * last_potentiometer / 1024);
				}
			}
			if (compareWithOverflow(time, recordedTime, 10000))
			{
				flags.init = 0;
				recordedTime = time;
				// Write time hysteris value
				if (flags.hysterisChanged)
				{
					flags.hysterisChanged = 0;
					eeprom_write_byte(&eepromContents.time_hysteris, hysterisTime);
				}
			}
		}
		// Main program run
		else if (flags.init == 0)
		{
			// presence detector by analog
			if (flags.analog)
			{
				absence_cnt = 0;
				flags.present = 1;
				flags.analog = 0;
			}
			// presence detector by IR
			else if ((PINB & (1 << MOVE_SENSOR)) != 0)
			{
				// presence detected. reset timer
				recordedTime = time;
				absence_cnt = 0;
				flags.present = 1;
			}
			else if (compareWithOverflow(time, recordedTime, 60000))
			{
				// nobody
				recordedTime = time;
				absence_cnt++;
				if (absence_cnt >= hysterisTime)
				{
					flags.present = 0;
				}
			}

			// photoresitor
			tmp = readAdcChannel(PHOTORESITOR);
			if (tmp > normalize0_1024(last_potentiometer + 75))
			{
				flags.light = 1;
			}
			else if (tmp < normalize0_1024(last_potentiometer - 75))
			{
				flags.light = 0;
			}

			// relay
			if (flags.present && !flags.light)
			{
				// relay on
				PORTB |= (1 << RELAY);
			}
			else
			{
				// relay off
				PORTB &= ~(1 << RELAY);
			}
		}

		// Display
		if (flags.init)
			output(time, hysterisTime);
		else if (flags.present)
			output(time, absDiff(32*last_potentiometer / 1024,  32*tmp /1024));
		else
			output(time, 0);
	}

	// Not reacheable
	return 1;
}
