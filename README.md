# -DIY-weighing-scale-
'''c
/*
 * Weighing scale with buttons - Nuvoton N79E352RADG (Keil C51)
 * HX711 + load cell + 16x2 LCD (8-bit) + 4 push buttons
 *
 * PINS
 *   HX711 DOUT -> P1.0 (pin 1)     HX711 SCK -> P1.1 (pin 2)
 *   LCD D0-D7  -> P2.0-P2.7        LCD RS -> P3.2 (pin 12)   LCD EN -> P3.3 (pin 13)
 *   SW1 TARE   -> P3.4 (pin 14)    SW2 UNIT -> P3.5 (pin 15)
 *   SW3 CAL    -> P3.6 (pin 16)    SW4 HOLD -> P3.7 (pin 17)
 *   Each button connects its pin to GND when pressed.
 *
 * BUTTONS
 *   TARE : set current weight to zero
 *   UNIT : cycle g -> kg -> lb
 *   HOLD : freeze / release the displayed weight
 *   CAL  : calibrate with a known weight (CAL_WEIGHT_G)
 *
 * No floating point is used, so the code stays small (chip has 8KB flash).
 */

#include <reg52.h>
#include <intrins.h>

/* Port mode registers (N79E352). 0,0 = quasi-bidirectional (internal pull-up). */
sfr P1M1 = 0xB3;
sfr P1M2 = 0xB4;
sfr P2M1 = 0xB5;
sfr P2M2 = 0xB6;
sfr P3M1 = 0x9E;
sfr P3M2 = 0x9F;

sbit HX_DOUT  = P1^0;
sbit HX_SCK   = P1^1;
sbit LCD_RS   = P3^2;
sbit LCD_EN   = P3^3;
sbit BTN_TARE = P3^4;
sbit BTN_UNIT = P3^5;
sbit BTN_CAL  = P3^6;
sbit BTN_HOLD = P3^7;
#define LCD_DATA P2

#define CAL_WEIGHT_G   500L    /* known weight you place during calibration */
#define DEFAULT_SCALE  420L    /* HX711 counts per gram (replace with your value) */
#define DELAY_LOOPS    550     /* 1 ms at 22.1184 MHz (4-clock core); adjust if needed */

enum { EV_NONE = 0, EV_TARE, EV_UNIT, EV_CAL, EV_HOLD };

static long offset = 0;
static long scale  = DEFAULT_SCALE;
static unsigned char unit = 0;        /* 0=g 1=kg 2=lb */
static bit hold = 0;
static unsigned char event = EV_NONE;
static unsigned char last_key = EV_NONE;

/* ---------------- delay ---------------- */
void delay_ms(unsigned int ms)
{
    unsigned int i, j;
    for (i = 0; i < ms; i++)
        for (j = 0; j < DELAY_LOOPS; j++)
            _nop_();
}

/* ---------------- LCD ---------------- */
void lcd_cmd(unsigned char c)
{
    LCD_RS = 0; LCD_DATA = c;
    LCD_EN = 1; delay_ms(1); LCD_EN = 0;
    delay_ms(2);
}

void lcd_char(unsigned char d)
{
    LCD_RS = 1; LCD_DATA = d;
    LCD_EN = 1; delay_ms(1); LCD_EN = 0;
    delay_ms(1);
}

void lcd_str(char *s) { while (*s) lcd_char(*s++); }

void lcd_goto(unsigned char row, unsigned char col)
{
    lcd_cmd((row == 0 ? 0x80 : 0xC0) + col);
}

void lcd_init(void)
{
    delay_ms(30);
    lcd_cmd(0x38);
    lcd_cmd(0x0C);
    lcd_cmd(0x06);
    lcd_cmd(0x01);
}

/* print a signed number with 'dec' decimals, right aligned in 'width' chars */
void lcd_fixed(long val, unsigned char dec, unsigned char width)
{
    char buf[14];
    unsigned char i = 0, k = 0, j;
    unsigned long u;
    bit neg = 0;

    if (val < 0) { neg = 1; u = (unsigned long)(-val); }
    else         { u = (unsigned long)val; }

    do {
        buf[i++] = '0' + (unsigned char)(u % 10);
        u /= 10;
        k++;
        if (k == dec) buf[i++] = '.';
    } while (u > 0 || k <= dec);

    if (neg) buf[i++] = '-';

    for (j = i; j < width; j++) lcd_char(' ');
    while (i > 0) lcd_char(buf[--i]);
}

/* ---------------- buttons ---------------- */
unsigned char read_key(void)
{
    if (!BTN_TARE) return EV_TARE;
    if (!BTN_UNIT) return EV_UNIT;
    if (!BTN_CAL)  return EV_CAL;
    if (!BTN_HOLD) return EV_HOLD;
    return EV_NONE;
}

/* call often; records one event per button press (debounced) */
void scan_buttons(void)
{
    unsigned char k = read_key();
    if (k != EV_NONE && last_key == EV_NONE) {
        delay_ms(20);                       /* debounce */
        if (read_key() == k) event = k;
    }
    last_key = k;
}

/* ---------------- HX711 ---------------- */
long hx711_read(void)
{
    unsigned long data = 0;
    unsigned char i;

    HX_SCK = 0;
    while (HX_DOUT) scan_buttons();         /* wait for data, keep buttons alive */

    for (i = 0; i < 24; i++) {
        HX_SCK = 1;
        _nop_();
        data <<= 1;
        HX_SCK = 0;
        if (HX_DOUT) data++;
    }
    HX_SCK = 1; _nop_(); HX_SCK = 0;        /* 25th pulse: channel A, gain 128 */

    if (data & 0x800000UL) data |= 0xFF000000UL;   /* sign extend */
    return (long)data;
}

long hx711_avg(unsigned char n)
{
    long sum = 0;
    unsigned char i;
    for (i = 0; i < n; i++) sum += hx711_read();
    return sum / n;
}

/* ---------------- calibration ---------------- */
void wait_cal_press(void)
{
    while (!BTN_CAL);       /* wait until released first */
    delay_ms(30);
    while (BTN_CAL);        /* wait for press */
    delay_ms(30);
    while (!BTN_CAL);       /* wait for release */
    delay_ms(30);
}

void calibrate(void)
{
    long diff;

    lcd_cmd(0x01);
    lcd_str("CALIBRATION");
    lcd_goto(1, 0); lcd_str("Empty, press CAL");
    wait_cal_press();
    offset = hx711_avg(10);

    lcd_cmd(0x01);
    lcd_str("Put 500 g, press");
    lcd_goto(1, 0); lcd_str("CAL when ready");
    wait_cal_press();

    diff = hx711_avg(10) - offset;
    if (diff > 0) {
        scale = diff / CAL_WEIGHT_G;
        if (scale < 1) scale = 1;
    }

    lcd_cmd(0x01);
    lcd_str("Cal done. Factor");
    lcd_goto(1, 0);
    lcd_fixed(scale, 0, 8);
    delay_ms(2500);         /* write this number into DEFAULT_SCALE to keep it */
    lcd_cmd(0x01);
    last_key = EV_NONE;
    event = EV_NONE;
}

/* ---------------- display ---------------- */
void show_weight(long raw)
{
    long v;                 /* weight in tenths of a gram */
    v = ((raw - offset) * 10L) / scale;
    if (v > -5 && v < 5) v = 0;             /* dead zone +/-0.5 g */

    lcd_goto(0, 0);
    lcd_str(hold ? "Weight   [HOLD] " : "Weight          ");

    lcd_goto(1, 0);
    if (unit == 0) {                         /* grams, 1 decimal */
        lcd_fixed(v, 1, 9);
        lcd_str(" g     ");
    } else if (unit == 1) {                  /* kilograms, 3 decimals */
        lcd_fixed(v / 10L, 3, 9);
        lcd_str(" kg    ");
    } else {                                 /* pounds, 2 decimals */
        lcd_fixed((v * 100L) / 4536L, 2, 9);
        lcd_str(" lb    ");
    }
}

/* ---------------- main ---------------- */
void main(void)
{
    long raw = 0;

    /* quasi-bidirectional mode on ports 1,2,3 (buttons get pull-ups) */
    P1M1 = 0; P1M2 = 0;
    P2M1 = 0; P2M2 = 0;
    P3M1 = 0; P3M2 = 0;
    HX_DOUT = 1;
    BTN_TARE = BTN_UNIT = BTN_CAL = BTN_HOLD = 1;

    lcd_init();
    lcd_str("Digital Scale");
    lcd_goto(1, 0); lcd_str("Keep scale empty");
    delay_ms(1500);
    offset = hx711_avg(10);                  /* tare at power-up */
    lcd_cmd(0x01);

    while (1) {
        raw = hx711_avg(2);                  /* buttons are scanned while waiting */

        switch (event) {
            case EV_TARE: offset = raw; hold = 0; break;
            case EV_UNIT: unit = (unit + 1) % 3; break;
            case EV_HOLD: hold = !hold; break;
            case EV_CAL:  calibrate(); break;
            default: break;
        }
        event = EV_NONE;

        if (!hold) show_weight(raw);
        else {
            lcd_goto(0, 0);
            lcd_str("Weight   [HOLD] ");
        }
    }
}
