//+------------------------------------------------------------------+
//| HammerPairManager.mq5                                            |
//| مدير استراتيجيات الشمعة المطرقة والمطرقة المعكوسة                 |
//| پلتفرم: MetaTrader 5 / MQL5                                      |
//| نوع حساب: Hedging                                                 |
//+------------------------------------------------------------------+
#property strict
#property version   "1.00"
#property description "مدیریت سیگنال چکش و چکش معکوس، سفارش‌های جفتی و بازیابی وضعیت"
#property description "تمام توضیحات ورودی‌ها و گزارش‌های برنامه به زبان فارسی است."

#include <Trade/Trade.mqh>

CTrade trade;

//====================================================================
// تنظیمات قابل تغییر توسط کاربر
//====================================================================

input group "========== تنظیمات عمومی =========="
input double InpLot                    = 0.02;   // حجم هر معامله
input ulong  InpMagic                  = 26101001; // شناسه اختصاصی برنامه
input int    InpMaxSpreadPoints        = 30;      // حداکثر اسپرد مجاز به پوینت
input int    InpDeviationPoints        = 20;      // حداکثر لغزش مجاز به پوینت
input int    InpTimerSeconds           = 1;       // فاصله بررسی تکمیلی بر حسب ثانیه
input bool   InpEnableHammer           = true;    // فعال بودن چکش معمولی
input bool   InpEnableInvertedHammer   = true;    // فعال بودن چکش معکوس
input bool   InpEnableBuy               = true;    // اجازه معاملات خرید
input bool   InpEnableSell              = true;    // اجازه معاملات فروش

input group "========== تنظیم شکل چکش =========="
input double InpMinBodyPercent         = 5.0;     // حداقل بدنه به درصد دامنه
input double InpMaxBodyPercent         = 35.0;    // حداکثر بدنه به درصد دامنه
input double InpMinShadowBodyRatio     = 2.0;     // حداقل نسبت سایه اصلی به بدنه
input double InpMaxOppositeShadowRatio = 0.50;    // حداکثر سایه مخالف نسبت به بدنه
input double InpBodyPositionPercent    = 35.0;    // محدوده قرارگیری بدنه در انتهای کندل، درصد
input bool   InpRequireCandleColor     = true;    // بررسی رنگ کندل

input group "========== تنظیم جفت معاملات =========="
input double InpLockFraction            = 0.50;   // کسری از سود معامله برنده برای حد ضرر جفت
input double InpBasketTakeProfitMoney   = 0.0;    // حد سود سبد به ارز حساب؛ صفر یعنی غیرفعال
input double InpBasketStopLossMoney     = 0.0;    // حد ضرر سبد به ارز حساب؛ صفر یعنی غیرفعال
input double InpMaxPairLossMoney        = 0.0;    // حداکثر زیان هر جفت؛ صفر یعنی غیرفعال

input group "========== عمر سفارش‌های معلق =========="
input double InpPendingLifetimeFactor   = 6.0;    // ضریب عمر سفارش معلق نسبت به تایم‌فریم
input int    InpPendingRetrySeconds     = 1;      // فاصله تلاش مجدد برای تکمیل جفت

input group "========== تایم‌فریم‌های فعال =========="
input bool InpTF_M1  = false;  // یک دقیقه
input bool InpTF_M2  = false;  // دو دقیقه
input bool InpTF_M3  = false;  // سه دقیقه
input bool InpTF_M4  = false;  // چهار دقیقه
input bool InpTF_M5  = true;   // پنج دقیقه
input bool InpTF_M6  = false;  // شش دقیقه
input bool InpTF_M10 = false;  // ده دقیقه
input bool InpTF_M12 = false;  // دوازده دقیقه
input bool InpTF_M15 = false;  // پانزده دقیقه
input bool InpTF_M20 = false;  // بیست دقیقه
input bool InpTF_M30 = false;  // سی دقیقه
input bool InpTF_H1  = true;   // یک ساعت
input bool InpTF_H2  = false;  // دو ساعت
input bool InpTF_H3  = false;  // سه ساعت
input bool InpTF_H4  = false;  // چهار ساعت
input bool InpTF_H6  = false;  // شش ساعت
input bool InpTF_H8  = false;  // هشت ساعت
input bool InpTF_H12 = false;  // دوازده ساعت
input bool InpTF_D1  = true;   // روزانه
input bool InpTF_W1  = false;  // هفتگی
input bool InpTF_MN1 = false;  // ماهانه

//====================================================================
// ساختارهای داده
//====================================================================

#define MAX_TFS 21
#define MAX_PAIRS 3000

enum ENUM_PAIR_STATE
{
   PAIR_WAITING_SECOND = 0,
   PAIR_BOTH_PLACED    = 1,
   PAIR_FIRST_WON      = 2,
   PAIR_SECOND_WON     = 3,
   PAIR_CLOSED         = 4
};

struct PairRecord
{
   string symbol;
   ENUM_TIMEFRAMES timeframe;
   datetime candle_time;
   string pair_id;
   int signal_type;
   int candle_color;
   double body_low;
   double body_high;
   double candle_low;
   double candle_high;
   double volume;
   double first_entry;
   double second_entry;
   double target_price;
   datetime expiry_time;
   ulong first_order;
   ulong second_order;
   ulong first_position;
   ulong second_position;
   ENUM_PAIR_STATE state;
   bool first_was_placed;
   bool second_was_placed;
   bool first_was_closed;
   bool second_was_closed;
   bool first_won;
   bool second_won;
   bool lock_applied;
   double first_profit;
   double second_profit;
   datetime last_retry;
};

struct SymbolTF
{
   string symbol;
   ENUM_TIMEFRAMES tf;
   datetime last_bar;
};

PairRecord pairs[MAX_PAIRS];
int pair_count=0;

SymbolTF configurations[];
int configuration_count=0;

string program_name="HammerPairManager";
string log_file="";
int log_handle=INVALID_HANDLE;
datetime last_log_day=0;
bool initialized=false;

//====================================================================
// توابع پایه و گزارش
//====================================================================

string TFName(ENUM_TIMEFRAMES tf)
{
   string s=EnumToString(tf);
   StringReplace(s,"PERIOD_","");
   return s;
}

string StateName(ENUM_PAIR_STATE state)
{
   switch(state)
   {
      case PAIR_WAITING_SECOND: return "انتظار برای جفت دوم";
      case PAIR_BOTH_PLACED:    return "هر دو سمت ثبت شده";
      case PAIR_FIRST_WON:      return "سمت اول برنده";
      case PAIR_SECOND_WON:     return "سمت دوم برنده";
      case PAIR_CLOSED:         return "بسته شده";
   }
   return "نامشخص";
}

void OpenDailyLog()
{
   datetime now=TimeLocal();
   MqlDateTime dt;
   TimeToStruct(now,dt);
   datetime day=StringToTime(StringFormat("%04d.%02d.%02d",dt.year,dt.mon,dt.day));

   if(log_handle!=INVALID_HANDLE && day==last_log_day)
      return;

   if(log_handle!=INVALID_HANDLE)
      FileClose(log_handle);

   log_file=StringFormat("%s_%04d-%02d-%02d.csv",
                         program_name,dt.year,dt.mon,dt.day);

   log_handle=FileOpen(log_file,
                       FILE_READ|FILE_WRITE|FILE_CSV|FILE_ANSI|
                       FILE_SHARE_READ,';');

   if(log_handle!=INVALID_HANDLE)
   {
      FileSeek(log_handle,0,SEEK_END);
      if(FileSize(log_handle)==0)
      {
         FileWrite(log_handle,"زمان","سطح","نماد","تایم‌فریم",
                   "شناسه جفت","رویداد","جزئیات");
      }
      last_log_day=day;
   }
}

void LogEvent(string level,string symbol,string tf,
              string pair_id,string event,string details)
{
   OpenDailyLog();

   string msg=StringFormat("[%s] %s | %s | %s | %s | %s",
      TimeToString(TimeLocal(),TIME_DATE|TIME_SECONDS),
      level,symbol,tf,event,details);

   Print(msg);

   if(log_handle!=INVALID_HANDLE)
   {
      FileWrite(log_handle,
         TimeToString(TimeLocal(),TIME_DATE|TIME_SECONDS),
         level,symbol,tf,pair_id,event,details);
      FileFlush(log_handle);
   }
}

string PairComment(const PairRecord &p,string side)
{
   string kind=(p.signal_type==1 ? "چکش" : "معکوس");
   string color=(p.candle_color>0 ? "سبز" : "قرمز");
   string dir=side;
   return StringFormat("HPM|%s|%s|%s|%s|%s",
                       p.pair_id,TFName(p.timeframe),kind,color,dir);
}

string PairKey(string symbol,ENUM_TIMEFRAMES tf,datetime candle_time)
{
   return StringFormat("%s_%d_%I64d",symbol,(int)tf,(long)candle_time);
}

bool IsOurComment(string comment)
{
   return StringFind(comment,"HPM|")==0;
}

double NormalizePrice(string symbol,double price)
{
   int digits=(int)SymbolInfoInteger(symbol,SYMBOL_DIGITS);
   return NormalizeDouble(price,digits);
}

double NormalizeVolume(string symbol,double volume)
{
   double minvol=SymbolInfoDouble(symbol,SYMBOL_VOLUME_MIN);
   double maxvol=SymbolInfoDouble(symbol,SYMBOL_VOLUME_MAX);
   double step=SymbolInfoDouble(symbol,SYMBOL_VOLUME_STEP);

   if(step<=0.0)
      return 0.0;

   volume=MathMax(minvol,MathMin(maxvol,volume));
   volume=MathFloor(volume/step+1e-8)*step;

   int precision=2;
   if(step<0.01) precision=3;
   if(step<0.001) precision=4;

   return NormalizeDouble(volume,precision);
}

bool SpreadAllowed(string symbol)
{
   MqlTick tick;
   if(!SymbolInfoTick(symbol,tick) || tick.ask<=0 || tick.bid<=0)
      return false;

   double point=SymbolInfoDouble(symbol,SYMBOL_POINT);
   if(point<=0.0) return false;

   double spread=(tick.ask-tick.bid)/point;
   return spread<=InpMaxSpreadPoints;
}

bool TradeEnvironmentAllowed()
{
   if(!TerminalInfoInteger(TERMINAL_CONNECTED))
      return false;
   if(!TerminalInfoInteger(TERMINAL_TRADE_ALLOWED))
      return false;
   if(!MQLInfoInteger(MQL_TRADE_ALLOWED))
      return false;
   return true;
}

//====================================================================
// تنظیم تایم‌فریم‌ها و نمادهای Market Watch
//====================================================================

void AddTimeframe(ENUM_TIMEFRAMES tf)
{
   int n=ArraySize(configurations);
   ArrayResize(configurations,n+1);
   configurations[n].symbol="";
   configurations[n].tf=tf;
   configurations[n].last_bar=0;
}

void BuildTimeframes()
{
   ArrayResize(configurations,0);

   if(InpTF_M1)  AddTimeframe(PERIOD_M1);
   if(InpTF_M2)  AddTimeframe(PERIOD_M2);
   if(InpTF_M3)  AddTimeframe(PERIOD_M3);
   if(InpTF_M4)  AddTimeframe(PERIOD_M4);
   if(InpTF_M5)  AddTimeframe(PERIOD_M5);
   if(InpTF_M6)  AddTimeframe(PERIOD_M6);
   if(InpTF_M10) AddTimeframe(PERIOD_M10);
   if(InpTF_M12) AddTimeframe(PERIOD_M12);
   if(InpTF_M15) AddTimeframe(PERIOD_M15);
   if(InpTF_M20) AddTimeframe(PERIOD_M20);
   if(InpTF_M30) AddTimeframe(PERIOD_M30);
   if(InpTF_H1)  AddTimeframe(PERIOD_H1);
   if(InpTF_H2)  AddTimeframe(PERIOD_H2);
   if(InpTF_H3)  AddTimeframe(PERIOD_H3);
   if(InpTF_H4)  AddTimeframe(PERIOD_H4);
   if(InpTF_H6)  AddTimeframe(PERIOD_H6);
   if(InpTF_H8)  AddTimeframe(PERIOD_H8);
   if(InpTF_H12) AddTimeframe(PERIOD_H12);
   if(InpTF_D1)  AddTimeframe(PERIOD_D1);
   if(InpTF_W1)  AddTimeframe(PERIOD_W1);
   if(InpTF_MN1) AddTimeframe(PERIOD_MN1);
}

int BuildConfigurations()
{
   configuration_count=0;
   int symbols=SymbolsTotal(true);
   int tfs=ArraySize(configurations);

   if(symbols<=0 || tfs<=0)
      return 0;

   ArrayResize(configurations,symbols*tfs);

   for(int s=0;s<symbols;s++)
   {
      string sym=SymbolName(s,true);
      if(sym=="") continue;

      for(int t=0;t<tfs;t++)
      {
         int idx=configuration_count++;
         configurations[idx].symbol=sym;
         configurations[idx].tf=configurations[t].tf;
         configurations[idx].last_bar=0;
      }
   }

   ArrayResize(configurations,configuration_count);
   return configuration_count;
}

//====================================================================
// تشخیص چکش و چکش معکوس
//====================================================================

bool DetectHammer(const MqlRates &bar,int &signal_type,int &color)
{
   double range=bar.high-bar.low;
   double body=MathAbs(bar.close-bar.open);

   if(range<=0.0 || body<=0.0)
      return false;

   double upper=bar.high-MathMax(bar.open,bar.close);
   double lower=MathMin(bar.open,bar.close)-bar.low;
   double body_pct=100.0*body/range;

   if(body_pct<InpMinBodyPercent || body_pct>InpMaxBodyPercent)
      return false;

   double safe_body=MathMax(body,_Point*0.1);
   bool hammer=(lower/safe_body>=InpMinShadowBodyRatio &&
                upper/safe_body<=InpMaxOppositeShadowRatio &&
                100.0*(MathMax(bar.open,bar.close)-bar.low)/range
                    >=100.0-InpBodyPositionPercent);

   bool inverted=(upper/safe_body>=InpMinShadowBodyRatio &&
                  lower/safe_body<=InpMaxOppositeShadowRatio &&
                  100.0*(bar.high-MathMin(bar.open,bar.close))/range
                    >=100.0-InpBodyPositionPercent);

   color=(bar.close>bar.open ? 1 : -1);

   if(InpRequireCandleColor && bar.close==bar.open)
      return false;

   if(hammer && InpEnableHammer)
   {
      signal_type=1;
      return true;
   }

   if(inverted && InpEnableInvertedHammer)
   {
      signal_type=2;
      return true;
   }

   return false;
}

//====================================================================
// ذخیره‌سازی پایدار
//====================================================================

string StateFileName()
{
   long login=AccountInfoInteger(ACCOUNT_LOGIN);
   return StringFormat("%s_%I64d_%I64d.bin",
                       program_name,login,(long)InpMagic);
}

void SaveState()
{
   int h=FileOpen(StateFileName(),
                  FILE_WRITE|FILE_BIN|FILE_COMMON);
   if(h==INVALID_HANDLE)
   {
      LogEvent("خطا","","","","ذخیره‌سازی",
               "باز کردن فایل وضعیت امکان‌پذیر نبود.");
      return;
   }

   FileWriteInteger(h,pair_count,INT_VALUE);

   for(int i=0;i<pair_count;i++)
   {
      PairRecord p=pairs[i];
      FileWriteString(h,p.symbol);
      FileWriteInteger(h,(int)p.timeframe,INT_VALUE);
      FileWriteLong(h,(long)p.candle_time);
      FileWriteString(h,p.pair_id);
      FileWriteInteger(h,p.signal_type,INT_VALUE);
      FileWriteInteger(h,p.candle_color,INT_VALUE);
      FileWriteDouble(h,p.body_low);
      FileWriteDouble(h,p.body_high);
      FileWriteDouble(h,p.candle_low);
      FileWriteDouble(h,p.candle_high);
      FileWriteDouble(h,p.volume);
      FileWriteDouble(h,p.first_entry);
      FileWriteDouble(h,p.second_entry);
      FileWriteDouble(h,p.target_price);
      FileWriteLong(h,(long)p.expiry_time);
      FileWriteLong(h,(long)p.first_order);
      FileWriteLong(h,(long)p.second_order);
      FileWriteLong(h,(long)p.first_position);
      FileWriteLong(h,(long)p.second_position);
      FileWriteInteger(h,(int)p.state,INT_VALUE);
      FileWriteInteger(h,p.first_was_placed?1:0,INT_VALUE);
      FileWriteInteger(h,p.second_was_placed?1:0,INT_VALUE);
      FileWriteInteger(h,p.first_was_closed?1:0,INT_VALUE);
      FileWriteInteger(h,p.second_was_closed?1:0,INT_VALUE);
      FileWriteInteger(h,p.first_won?1:0,INT_VALUE);
      FileWriteInteger(h,p.second_won?1:0,INT_VALUE);
      FileWriteInteger(h,p.lock_applied?1:0,INT_VALUE);
      FileWriteDouble(h,p.first_profit);
      FileWriteDouble(h,p.second_profit);
      FileWriteLong(h,(long)p.last_retry);
   }

   FileFlush(h);
   FileClose(h);
}

bool LoadState()
{
   int h=FileOpen(StateFileName(),FILE_READ|FILE_BIN|FILE_COMMON);
   if(h==INVALID_HANDLE)
      return false;

   int count=FileReadInteger(h,INT_VALUE);
   if(count<0 || count>MAX_PAIRS)
   {
      FileClose(h);
      return false;
   }

   pair_count=0;

   for(int i=0;i<count;i++)
   {
      if(FileIsEnding(h)) break;

      PairRecord p;
      ZeroMemory(p);

      p.symbol=FileReadString(h);
      p.timeframe=(ENUM_TIMEFRAMES)FileReadInteger(h,INT_VALUE);
      p.candle_time=(datetime)FileReadLong(h);
      p.pair_id=FileReadString(h);
      p.signal_type=FileReadInteger(h,INT_VALUE);
      p.candle_color=FileReadInteger(h,INT_VALUE);
      p.body_low=FileReadDouble(h);
      p.body_high=FileReadDouble(h);
      p.candle_low=FileReadDouble(h);
      p.candle_high=FileReadDouble(h);
      p.volume=FileReadDouble(h);
      p.first_entry=FileReadDouble(h);
      p.second_entry=FileReadDouble(h);
      p.target_price=FileReadDouble(h);
      p.expiry_time=(datetime)FileReadLong(h);
      p.first_order=(ulong)FileReadLong(h);
      p.second_order=(ulong)FileReadLong(h);
      p.first_position=(ulong)FileReadLong(h);
      p.second_position=(ulong)FileReadLong(h);
      p.state=(ENUM_PAIR_STATE)FileReadInteger(h,INT_VALUE);
      p.first_was_placed=(FileReadInteger(h,INT_VALUE)!=0);
      p.second_was_placed=(FileReadInteger(h,INT_VALUE)!=0);
      p.first_was_closed=(FileReadInteger(h,INT_VALUE)!=0);
      p.second_was_closed=(FileReadInteger(h,INT_VALUE)!=0);
      p.first_won=(FileReadInteger(h,INT_VALUE)!=0);
      p.second_won=(FileReadInteger(h,INT_VALUE)!=0);
      p.lock_applied=(FileReadInteger(h,INT_VALUE)!=0);
      p.first_profit=FileReadDouble(h);
      p.second_profit=FileReadDouble(h);
      p.last_retry=(datetime)FileReadLong(h);

      pairs[pair_count++]=p;
   }

   FileClose(h);
   return true;
}

//====================================================================
// جست‌وجوی وضعیت و جلوگیری از استفاده مجدد از کندل
//====================================================================

int FindPair(string key)
{
   for(int i=0;i<pair_count;i++)
      if(pairs[i].pair_id==key)
         return i;
   return -1;
}

bool HasUnfinishedPair(string symbol,ENUM_TIMEFRAMES tf)
{
   for(int i=0;i<pair_count;i++)
   {
      if(pairs[i].symbol==symbol &&
         pairs[i].timeframe==tf &&
         pairs[i].state!=PAIR_CLOSED &&
         pairs[i].state!=PAIR_FIRST_WON &&
         pairs[i].state!=PAIR_SECOND_WON)
         return true;
   }
   return false;
}

int CreatePair(string symbol,ENUM_TIMEFRAMES tf,
               datetime candle_time,int signal_type,int color,
               double body_low,double body_high,
               double candle_low,double candle_high)
{
   if(pair_count>=MAX_PAIRS)
   {
      LogEvent("خطا",symbol,TFName(tf),"","ساخت جفت",
               "ظرفیت رکوردهای جفت پر شده است.");
      return -1;
   }

   string key=PairKey(symbol,tf,candle_time);
   if(FindPair(key)>=0)
      return -1;

   PairRecord p;
   ZeroMemory(p);

   p.symbol=symbol;
   p.timeframe=tf;
   p.candle_time=candle_time;
   p.pair_id=key;
   p.signal_type=signal_type;
   p.candle_color=color;
   p.body_low=body_low;
   p.body_high=body_high;
   p.candle_low=candle_low;
   p.candle_high=candle_high;
   p.volume=NormalizeVolume(symbol,InpLot);
   p.expiry_time=candle_time+
                 (datetime)(PeriodSeconds(tf)*InpPendingLifetimeFactor);
   p.state=PAIR_WAITING_SECOND;
   p.last_retry=0;

   pairs[pair_count]=p;
   pair_count++;

   SaveState();

   LogEvent("اطلاعات",symbol,TFName(tf),key,
            "سیگنال شناسایی شد",
            StringFormat("%s، رنگ %s، بدنه %.5f تا %.5f",
            signal_type==1?"چکش":"چکش معکوس",
            color>0?"سبز":"قرمز",body_low,body_high));

   return pair_count-1;
}

//====================================================================
// ثبت سفارش‌ها
//====================================================================

bool SelectPendingByTicket(ulong ticket)
{
   if(ticket==0) return false;
   return OrderSelect(ticket);
}

bool PlacePending(PairRecord &p,bool first,ENUM_ORDER_TYPE type,
                  double price,double tp)
{
   if(!TradeEnvironmentAllowed())
      return false;

   if(!SpreadAllowed(p.symbol))
      return false;

   MqlTick tick;
   if(!SymbolInfoTick(p.symbol,tick))
      return false;

   double point=SymbolInfoDouble(p.symbol,SYMBOL_POINT);
   int stops=(int)SymbolInfoInteger(p.symbol,SYMBOL_TRADE_STOPS_LEVEL);
   double min_distance=stops*point;

   price=NormalizePrice(p.symbol,price);
   tp=NormalizePrice(p.symbol,tp);

   if(type==ORDER_TYPE_BUY_STOP && price<tick.ask+min_distance)
      return false;
   if(type==ORDER_TYPE_SELL_STOP && price>tick.bid-min_distance)
      return false;
   if(type==ORDER_TYPE_BUY_LIMIT && price>tick.ask-min_distance)
      return false;
   if(type==ORDER_TYPE_SELL_LIMIT && price<tick.bid+min_distance)
      return false;

   if(type==ORDER_TYPE_BUY_LIMIT || type==ORDER_TYPE_BUY_STOP)
   {
      if(tp<=price || tp<=0.0)
         return false;
   }
   else
   {
      if(tp>=price || tp<=0.0)
         return false;
   }

   datetime expiration=p.expiry_time;
   ENUM_ORDER_TYPE_TIME time_type=ORDER_TIME_SPECIFIED;

   if(expiration<=TimeCurrent())
   {
      expiration=TimeCurrent()+PeriodSeconds(p.timeframe);
   }

   trade.SetExpertMagicNumber(InpMagic);
   trade.SetDeviationInPoints(InpDeviationPoints);
   trade.SetTypeFillingBySymbol(p.symbol);

   string side=first?"A":"B";
   string comment=PairComment(p,side);

   bool ok=false;

   switch(type)
   {
      case ORDER_TYPE_BUY_LIMIT:
         ok=trade.BuyLimit(p.volume,price,p.symbol,0.0,tp,
                           time_type,expiration,comment);
         break;
      case ORDER_TYPE_SELL_LIMIT:
         ok=trade.SellLimit(p.volume,price,p.symbol,0.0,tp,
                            time_type,expiration,comment);
         break;
      case ORDER_TYPE_BUY_STOP:
         ok=trade.BuyStop(p.volume,price,p.symbol,0.0,tp,
                          time_type,expiration,comment);
         break;
      case ORDER_TYPE_SELL_STOP:
         ok=trade.SellStop(p.volume,price,p.symbol,0.0,tp,
                           time_type,expiration,comment);
         break;
      default:
         return false;
   }

   if(!ok)
   {
      LogEvent("هشدار",p.symbol,TFName(p.timeframe),p.pair_id,
               "ثبت سفارش ناموفق",
               StringFormat("%s، کد %u، %s",
                  side,trade.ResultRetcode(),
                  trade.ResultRetcodeDescription()));
      return false;
   }

   ulong ticket=trade.ResultOrder();

   if(first)
   {
      p.first_order=ticket;
      p.first_was_placed=true;
      p.first_entry=price;
   }
   else
   {
      p.second_order=ticket;
      p.second_was_placed=true;
      p.second_entry=price;
   }

   LogEvent("اطلاعات",p.symbol,TFName(p.timeframe),p.pair_id,
            "سفارش معلق ثبت شد",
            StringFormat("سمت %s، نوع %s، قیمت %.5f، حد سود %.5f",
                         side,EnumToString(type),price,tp));

   SaveState();
   return true;
}

//====================================================================
// ورود فوری و ثبت جفت مخالف
//====================================================================

bool OpenMarket(PairRecord &p,bool first,ENUM_ORDER_TYPE type,
                double tp)
{
   if(!TradeEnvironmentAllowed() || !SpreadAllowed(p.symbol))
      return false;

   MqlTick tick;
   if(!SymbolInfoTick(p.symbol,tick))
      return false;

   tp=NormalizePrice(p.symbol,tp);
   trade.SetExpertMagicNumber(InpMagic);
   trade.SetDeviationInPoints(InpDeviationPoints);
   trade.SetTypeFillingBySymbol(p.symbol);

   string side=first?"A":"B";
   string comment=PairComment(p,side);
   bool ok=false;

   if(type==ORDER_TYPE_BUY)
   {
      if(!InpEnableBuy) return false;
      if(tp<=tick.ask) return false;
      ok=trade.Buy(p.volume,p.symbol,0.0,0.0,tp,comment);
   }
   else if(type==ORDER_TYPE_SELL)
   {
      if(!InpEnableSell) return false;
      if(tp>=tick.bid) return false;
      ok=trade.Sell(p.volume,p.symbol,0.0,0.0,tp,comment);
   }

   if(!ok)
   {
      LogEvent("هشدار",p.symbol,TFName(p.timeframe),p.pair_id,
               "ورود فوری ناموفق",
               StringFormat("%s، کد %u، %s",
                  side,trade.ResultRetcode(),
                  trade.ResultRetcodeDescription()));
      return false;
   }

   ulong deal=trade.ResultDeal();

   if(first)
   {
      p.first_was_placed=true;
      p.first_entry=(type==ORDER_TYPE_BUY?tick.ask:tick.bid);
      p.first_position=deal;
   }
   else
   {
      p.second_was_placed=true;
      p.second_entry=(type==ORDER_TYPE_BUY?tick.ask:tick.bid);
      p.second_position=deal;
   }

   LogEvent("اطلاعات",p.symbol,TFName(p.timeframe),p.pair_id,
            "معامله بازار ثبت شد",
            StringFormat("سمت %s، جهت %s، حد سود %.5f",
                         side,EnumToString(type),tp));

   SaveState();
   return true;
}

//====================================================================
// ایجاد منطق دو سمت جفت
//====================================================================

void TryBuildPair(int idx)
{
   if(idx<0 || idx>=pair_count) return;

   PairRecord p=pairs[idx];

   if(p.state==PAIR_CLOSED ||
      p.state==PAIR_FIRST_WON ||
      p.state==PAIR_SECOND_WON)
      return;

   if(p.last_retry>0 &&
      TimeCurrent()-p.last_retry<InpPendingRetrySeconds)
      return;

   p.last_retry=TimeCurrent();

   if(!TradeEnvironmentAllowed())
   {
      pairs[idx]=p;
      return;
   }

   if(!SpreadAllowed(p.symbol))
   {
      pairs[idx]=p;
      return;
   }

   MqlTick tick;
   if(!SymbolInfoTick(p.symbol,tick))
   {
      pairs[idx]=p;
      return;
   }

   double point=SymbolInfoDouble(p.symbol,SYMBOL_POINT);
   double spread=tick.ask-tick.bid;

   // نوع 1 = چکش؛ نوع 2 = چکش معکوس
   // بدنه سبز: بای روی پایین بدنه، جفت مخالف سل
   // بدنه قرمز: بای فوری و سل پس از افزایش به اندازه اسپرد
   // معکوس سبز: سل فوری و بای جفتی
   // معکوس قرمز: سل لیمیت روی بالای بدنه

   if(p.signal_type==1 && p.candle_color>0)
   {
      if(!p.first_was_placed)
      {
         ENUM_ORDER_TYPE t=(p.body_low<tick.ask?
                            ORDER_TYPE_BUY_LIMIT:ORDER_TYPE_BUY_STOP);
         if(PlacePending(p,true,t,p.body_low,p.body_high))
            p.first_was_placed=true;
      }

      if(!p.second_was_placed)
      {
         ENUM_ORDER_TYPE t=(p.body_low>tick.bid?
                            ORDER_TYPE_SELL_LIMIT:ORDER_TYPE_SELL_STOP);
         if(PlacePending(p,false,t,p.body_low,p.body_low-spread))
            p.second_was_placed=true;
      }
   }
   else if(p.signal_type==1 && p.candle_color<0)
   {
      if(!p.first_was_placed)
      {
         if(OpenMarket(p,true,ORDER_TYPE_BUY,p.body_high))
            p.first_was_placed=true;
      }

      if(!p.second_was_placed &&
         tick.ask>=p.first_entry+spread)
      {
         if(OpenMarket(p,false,ORDER_TYPE_SELL,p.body_high))
            p.second_was_placed=true;
      }
   }
   else if(p.signal_type==2 && p.candle_color>0)
   {
      if(!p.first_was_placed)
      {
         if(OpenMarket(p,true,ORDER_TYPE_SELL,p.body_low))
            p.first_was_placed=true;
      }

      if(!p.second_was_placed)
      {
         ENUM_ORDER_TYPE t=(p.body_high>tick.bid?
                            ORDER_TYPE_SELL_LIMIT:ORDER_TYPE_SELL_STOP);
         if(PlacePending(p,false,t,p.body_high,p.body_low))
            p.second_was_placed=true;
      }
   }
   else if(p.signal_type==2 && p.candle_color<0)
   {
      if(!p.first_was_placed)
      {
         ENUM_ORDER_TYPE t=(p.body_high>tick.bid?
                            ORDER_TYPE_SELL_LIMIT:ORDER_TYPE_SELL_STOP);
         if(PlacePending(p,true,t,p.body_high,p.body_low))
            p.first_was_placed=true;
      }

      if(!p.second_was_placed)
      {
         ENUM_ORDER_TYPE t=(p.body_high<tick.ask?
                            ORDER_TYPE_BUY_LIMIT:ORDER_TYPE_BUY_STOP);
         if(PlacePending(p,false,t,p.body_high,p.body_high+spread))
            p.second_was_placed=true;
      }
   }

   if(p.first_was_placed && p.second_was_placed)
      p.state=PAIR_BOTH_PLACED;

   pairs[idx]=p;
   SaveState();
}

//====================================================================
// مدیریت پوزیشن‌ها، تاریخچه و جفت‌های قبلی
//====================================================================

bool FindPositionByComment(string comment,ulong &ticket,
                           ENUM_POSITION_TYPE &type,double &volume,
                           double &profit,double &open_price)
{
   for(int i=PositionsTotal()-1;i>=0;i--)
   {
      ulong t=PositionGetTicket(i);
      if(t==0) continue;

      if(!PositionSelectByTicket(t))
         continue;

      if(PositionGetInteger(POSITION_MAGIC)!=(long)InpMagic)
         continue;

      if(PositionGetString(POSITION_COMMENT)!=comment)
         continue;

      ticket=t;
      type=(ENUM_POSITION_TYPE)PositionGetInteger(POSITION_TYPE);
      volume=PositionGetDouble(POSITION_VOLUME);
      profit=PositionGetDouble(POSITION_PROFIT)+
             PositionGetDouble(POSITION_SWAP);
      open_price=PositionGetDouble(POSITION_PRICE_OPEN);
      return true;
   }
   return false;
}

bool ModifyPositionSL(ulong ticket,double sl,double tp)
{
   if(!PositionSelectByTicket(ticket))
      return false;

   string symbol=PositionGetString(POSITION_SYMBOL);
   ENUM_POSITION_TYPE type=(ENUM_POSITION_TYPE)
                            PositionGetInteger(POSITION_TYPE);

   MqlTick tick;
   if(!SymbolInfoTick(symbol,tick))
      return false;

   double point=SymbolInfoDouble(symbol,SYMBOL_POINT);
   int stops=(int)SymbolInfoInteger(symbol,SYMBOL_TRADE_STOPS_LEVEL);
   double min_distance=stops*point;

   sl=NormalizePrice(symbol,sl);
   tp=NormalizePrice(symbol,tp);

   if(type==POSITION_TYPE_BUY)
   {
      if(sl>=tick.bid-min_distance) return false;
      if(tp>0 && tp<=tick.bid) tp=0;
   }
   else
   {
      if(sl<=tick.ask+min_distance) return false;
      if(tp>0 && tp>=tick.ask) tp=0;
   }

   trade.SetExpertMagicNumber(InpMagic);
   bool ok=trade.PositionModify(ticket,sl,tp);

   if(!ok)
   {
      LogEvent("هشدار",symbol,"","",
               "تغییر حد ضرر ناموفق",
               StringFormat("تیکت %I64u، کد %u، %s",
                 ticket,trade.ResultRetcode(),
                 trade.ResultRetcodeDescription()));
   }

   return ok;
}

double ClosedProfitByPositionIdentifier(ulong position_id)
{
   if(position_id==0) return 0.0;

   datetime from=TimeCurrent()-86400*60;
   datetime to=TimeCurrent();

   if(!HistorySelect(from,to))
      return 0.0;

   double profit=0.0;

   for(int i=0;i<HistoryDealsTotal();i++)
   {
      ulong deal=HistoryDealGetTicket(i);
      if(deal==0) continue;

      if((ulong)HistoryDealGetInteger(deal,DEAL_POSITION_ID)!=position_id)
         continue;

      if((ulong)HistoryDealGetInteger(deal,DEAL_MAGIC)!=InpMagic)
         continue;

      profit+=HistoryDealGetDouble(deal,DEAL_PROFIT);
      profit+=HistoryDealGetDouble(deal,DEAL_SWAP);
      profit+=HistoryDealGetDouble(deal,DEAL_COMMISSION);
   }

   return profit;
}

void ManagePair(int idx)
{
   if(idx<0 || idx>=pair_count) return;

   PairRecord p=pairs[idx];
   if(p.state==PAIR_CLOSED) return;

   string ca=PairComment(p,"A");
   string cb=PairComment(p,"B");

   ulong ta=0,tb=0;
   ENUM_POSITION_TYPE typea,typeb;
   double vola=0,volb=0,profa=0,profb=0,opena=0,openb=0;

   bool hasa=FindPositionByComment(ca,ta,typea,vola,profa,opena);
   bool hasb=FindPositionByComment(cb,tb,typeb,volb,profb,openb);

   if(hasa) p.first_position=ta;
   if(hasb) p.second_position=tb;

   // تشخیص برنده فقط پس از فعال شدن هر دو سمت
   if(hasa && hasb)
   {
      p.state=PAIR_BOTH_PLACED;

      if(profa>0.0 && !p.lock_applied)
      {
         p.first_won=true;
         p.first_profit=profa;
         p.state=PAIR_FIRST_WON;

         double distance=MathAbs(opena-p.first_entry);
         double lock_distance=distance*InpLockFraction;
         double sl=(typeb==POSITION_TYPE_SELL?
                    openb+lock_distance:openb-lock_distance);

         if(ModifyPositionSL(tb,sl,0.0))
         {
            p.lock_applied=true;
            LogEvent("اطلاعات",p.symbol,TFName(p.timeframe),p.pair_id,
                     "حد ضرر جفت تنظیم شد",
                     StringFormat("برنده A، سود شناور %.2f، حد ضرر B %.5f",
                                  profa,sl));
         }
      }
      else if(profb>0.0 && !p.lock_applied)
      {
         p.second_won=true;
         p.second_profit=profb;
         p.state=PAIR_SECOND_WON;

         double distance=MathAbs(openb-p.second_entry);
         double lock_distance=distance*InpLockFraction;
         double sl=(typea==POSITION_TYPE_SELL?
                    opena+lock_distance:opena-lock_distance);

         if(ModifyPositionSL(ta,sl,0.0))
         {
            p.lock_applied=true;
            LogEvent("اطلاعات",p.symbol,TFName(p.timeframe),p.pair_id,
                     "حد ضرر جفت تنظیم شد",
                     StringFormat("برنده B، سود شناور %.2f، حد ضرر A %.5f",
                                  profb,sl));
         }
      }
   }

   // حذف سفارش‌های معلق منقضی‌شده؛ جفت مصرف‌شده باقی می‌ماند
   if(TimeCurrent()>=p.expiry_time)
   {
      ulong orders[2]={p.first_order,p.second_order};
      for(int j=0;j<2;j++)
      {
         ulong order=orders[j];
         if(order>0 && OrderSelect(order))
         {
            ENUM_ORDER_TYPE ot=(ENUM_ORDER_TYPE)OrderGetInteger(ORDER_TYPE);
            if(ot==ORDER_TYPE_BUY_LIMIT || ot==ORDER_TYPE_SELL_LIMIT ||
               ot==ORDER_TYPE_BUY_STOP || ot==ORDER_TYPE_SELL_STOP ||
               ot==ORDER_TYPE_BUY_STOP_LIMIT || ot==ORDER_TYPE_SELL_STOP_LIMIT)
            {
               trade.SetExpertMagicNumber(InpMagic);
               if(trade.OrderDelete(order))
                  LogEvent("اطلاعات",p.symbol,TFName(p.timeframe),p.pair_id,
                           "سفارش منقضی شد",
                           StringFormat("تیکت %I64u حذف شد",order));
            }
         }
      }
   }

   // ثبت وضعیت جفت در پایان
   if(!hasa && !hasb &&
      !p.first_was_placed && !p.second_was_placed)
      p.state=PAIR_CLOSED;

   pairs[idx]=p;
}

//====================================================================
// کنترل سبد معاملاتی
//====================================================================

double BasketFloatingProfit()
{
   double total=0.0;

   for(int i=PositionsTotal()-1;i>=0;i--)
   {
      ulong ticket=PositionGetTicket(i);
      if(ticket==0 || !PositionSelectByTicket(ticket))
         continue;

      if(PositionGetInteger(POSITION_MAGIC)!=(long)InpMagic)
         continue;

      total+=PositionGetDouble(POSITION_PROFIT);
      total+=PositionGetDouble(POSITION_SWAP);
   }

   return total;
}

void CloseAllOurPositionsAndOrders(string reason)
{
   trade.SetExpertMagicNumber(InpMagic);

   for(int i=PositionsTotal()-1;i>=0;i--)
   {
      ulong ticket=PositionGetTicket(i);
      if(ticket==0 || !PositionSelectByTicket(ticket))
         continue;

      if(PositionGetInteger(POSITION_MAGIC)!=(long)InpMagic)
         continue;

      string sym=PositionGetString(POSITION_SYMBOL);
      if(trade.PositionClose(ticket))
         LogEvent("هشدار",sym,"","","بستن سبد",reason);
   }

   for(int i=OrdersTotal()-1;i>=0;i--)
   {
      ulong ticket=OrderGetTicket(i);
      if(ticket==0 || !OrderSelect(ticket))
         continue;

      if((ulong)OrderGetInteger(ORDER_MAGIC)!=InpMagic)
         continue;

      string sym=OrderGetString(ORDER_SYMBOL);
      if(trade.OrderDelete(ticket))
         LogEvent("هشدار",sym,"","","حذف سفارش",reason);
   }

   for(int i=0;i<pair_count;i++)
      pairs[i].state=PAIR_CLOSED;

   SaveState();
}

void CheckBasket()
{
   double profit=BasketFloatingProfit();

   if(InpBasketTakeProfitMoney>0.0 &&
      profit>=InpBasketTakeProfitMoney)
   {
      CloseAllOurPositionsAndOrders(
         StringFormat("حد سود سبد %.2f فعال شد",profit));
   }
   else if(InpBasketStopLossMoney>0.0 &&
           profit<=-InpBasketStopLossMoney)
   {
      CloseAllOurPositionsAndOrders(
         StringFormat("حد ضرر سبد %.2f فعال شد",profit));
   }
}

//====================================================================
// بازسازی اطلاعات از سفارش‌ها و پوزیشن‌های زنده
//====================================================================

void ReconcileLiveOrders()
{
   for(int i=OrdersTotal()-1;i>=0;i--)
   {
      ulong ticket=OrderGetTicket(i);
      if(ticket==0 || !OrderSelect(ticket))
         continue;

      if((ulong)OrderGetInteger(ORDER_MAGIC)!=InpMagic)
         continue;

      string comment=OrderGetString(ORDER_COMMENT);
      if(!IsOurComment(comment))
         continue;

      string parts[];
      if(StringSplit(comment,'|',parts)<6)
         continue;

      string key=parts[1];
      int idx=FindPair(key);
      if(idx<0) continue;

      ENUM_ORDER_TYPE type=(ENUM_ORDER_TYPE)OrderGetInteger(ORDER_TYPE);
      if(parts[5]=="A")
      {
         pairs[idx].first_order=ticket;
         pairs[idx].first_was_placed=true;
      }
      else if(parts[5]=="B")
      {
         pairs[idx].second_order=ticket;
         pairs[idx].second_was_placed=true;
      }
   }

   SaveState();
}

//====================================================================
// اسکن کندل‌های بسته‌شده
//====================================================================

void ScanConfiguration(int idx)
{
   if(idx<0 || idx>=configuration_count) return;

   string symbol=configurations[idx].symbol;
   ENUM_TIMEFRAMES tf=configurations[idx].tf;

   if(symbol=="") return;

   MqlRates rates[3];
   ArraySetAsSeries(rates,true);

   int copied=CopyRates(symbol,tf,0,3,rates);
   if(copied<3) return;

   datetime current_bar=rates[0].time;

   if(configurations[idx].last_bar==0)
   {
      configurations[idx].last_bar=current_bar;
      return;
   }

   if(current_bar==configurations[idx].last_bar)
      return;

   configurations[idx].last_bar=current_bar;

   MqlRates signal_bar=rates[1];
   int signal_type=0,color=0;

   if(!DetectHammer(signal_bar,signal_type,color))
      return;

   // فرصت مستقل برای هر کندل؛ هر سیگنال فقط یک بار ثبت می‌شود
   string key=PairKey(symbol,tf,signal_bar.time);
   if(FindPair(key)>=0)
      return;

   double body_low=MathMin(signal_bar.open,signal_bar.close);
   double body_high=MathMax(signal_bar.open,signal_bar.close);

   int pair_idx=CreatePair(symbol,tf,signal_bar.time,
                           signal_type,color,body_low,body_high,
                           signal_bar.low,signal_bar.high);

   if(pair_idx>=0)
      TryBuildPair(pair_idx);
}

//====================================================================
// رویدادهای اصلی اکسپرت
//====================================================================

int OnInit()
{
   if(InpLot<=0.0 || InpLockFraction<0.0 ||
      InpLockFraction>1.0 || InpPendingLifetimeFactor<=0.0 ||
      InpTimerSeconds<1 || InpMinBodyPercent<0.0 ||
      InpMaxBodyPercent>100.0 ||
      InpMinBodyPercent>InpMaxBodyPercent)
   {
      Print("تنظیمات ورودی نامعتبر است.");
      return INIT_PARAMETERS_INCORRECT;
   }

   if(AccountInfoInteger(ACCOUNT_MARGIN_MODE)!=ACCOUNT_MARGIN_MODE_RETAIL_HEDGING)
   {
      Print("این برنامه برای حساب Hedging طراحی شده است.");
      return INIT_FAILED;
   }

   trade.SetExpertMagicNumber(InpMagic);
   trade.SetDeviationInPoints(InpDeviationPoints);

   BuildTimeframes();
   BuildConfigurations();

   LoadState();
   ReconcileLiveOrders();

   EventSetTimer(InpTimerSeconds);
   OpenDailyLog();

   initialized=true;

   LogEvent("اطلاعات","","","","شروع برنامه",
            StringFormat("نمادها و تایم‌فریم‌های فعال: %d",
                         configuration_count));

   return INIT_SUCCEEDED;
}

void OnDeinit(const int reason)
{
   SaveState();

   if(log_handle!=INVALID_HANDLE)
   {
      FileFlush(log_handle);
      FileClose(log_handle);
      log_handle=INVALID_HANDLE;
   }

   EventKillTimer();
}

void OnTick()
{
   if(!initialized) return;

   // مدیریت سریع جفت‌ها و سبد در هر تیک
   for(int i=0;i<pair_count;i++)
   {
      TryBuildPair(i);
      ManagePair(i);
   }

   CheckBasket();

   // اسکن سیگنال‌ها روی همه نمادها و تایم‌فریم‌ها
   for(int j=0;j<configuration_count;j++)
      ScanConfiguration(j);
}

void OnTimer()
{
   if(!initialized) return;

   OpenDailyLog();

   // تلاش مجدد برای جفت‌های ناقص حتی بعد از بسته‌شدن کندل
   for(int i=0;i<pair_count;i++)
   {
      TryBuildPair(i);
      ManagePair(i);
   }

   CheckBasket();
   ReconcileLiveOrders();
   SaveState();
}

//====================================================================
// پیگیری معاملات و خروج‌ها
//====================================================================

void OnTradeTransaction(const MqlTradeTransaction &trans,
                        const MqlTradeRequest &request,
                        const MqlTradeResult &result)
{
   if(trans.type==TRADE_TRANSACTION_DEAL_ADD)
   {
      if(!HistoryDealSelect(trans.deal))
         return;

      if((ulong)HistoryDealGetInteger(trans.deal,DEAL_MAGIC)!=InpMagic)
         return;

      string symbol=HistoryDealGetString(trans.deal,DEAL_SYMBOL);
      string comment=HistoryDealGetString(trans.deal,DEAL_COMMENT);

      LogEvent("اطلاعات",symbol,"","",
               "تغییر معامله",
               StringFormat("دیل %I64u، نوع %d، سود %.2f، توضیح %s",
                  trans.deal,
                  (int)HistoryDealGetInteger(trans.deal,DEAL_ENTRY),
                  HistoryDealGetDouble(trans.deal,DEAL_PROFIT),
                  comment));

      SaveState();
   }
}