//+------------------------------------------------------------------+
//|                                     N8N_Webhook_Trader.mq5 |
//|             Polls n8n for trade signals and executes them        |
//+------------------------------------------------------------------+
#property copyright "AI Project"
#property link      "https://spottertradeai.com"
#property version   "2.01" // Corrected WebRequest call
#property strict

#include <Trade\Trade.mqh>
#include <JAson.mqh>     // uses CJAVal

// --- Inputs ---
input string   N8N_URL       = "https://your-n8n-host/webhook/xyz"; // n8n GET/POST endpoint that returns JSON
input int      POLL_SECS     = ;           // how often to poll n8n (seconds)
input double   DefaultLots = 0.01;        // lot size
input ulong    MagicNumber = 1337;        // magic

// --- Globals ---
CTrade trade;

//--- forward
bool FetchJSON(string &json_string);
void ProcessWebhook(const string json_string);

//+------------------------------------------------------------------+
//| Expert initialization                                            |
//+------------------------------------------------------------------+
int OnInit()
{
   trade.SetExpertMagicNumber(MagicNumber);
   // IMPORTANT: In MT5: Tools → Options → Expert Advisors → Allow WebRequest for listed URL(s)
   // Add the base URL of your n8n server (e.g., https://your-n8n-host)
   EventSetTimer(POLL_SECS);
   PrintFormat("N8N Webhook Trader started. Polling %s every %d second(s).", N8N_URL, POLL_SECS);
   return(INIT_SUCCEEDED);
}

//+------------------------------------------------------------------+
//| Expert deinitialization                                          |
//+------------------------------------------------------------------+
void OnDeinit(const int reason)
{
   EventKillTimer();
   Print("N8N Webhook Trader stopped.");
}

//+------------------------------------------------------------------+
//| Timer: poll n8n                                                  |
//+------------------------------------------------------------------+
void OnTimer()
{
   string json_string;
   if(FetchJSON(json_string) && StringLen(json_string) > 0)
   {
      ProcessWebhook(json_string);
   }
}

//+------------------------------------------------------------------+
//| Fetch JSON from n8n via WebRequest                               |
//+------------------------------------------------------------------+
bool FetchJSON(string &json_string)
{
   string headers; // Empty headers for this simple request
   char   body[];         // Empty body for a GET request
   char   result[];
   string result_headers;
   int    timeout_ms = 8000;

   // CORRECTED: The WebRequest call now uses the correct parameters.
   // We pass the empty 'body' array instead of NULL.
   int status = WebRequest("GET", N8N_URL, headers, timeout_ms, body, result, result_headers);

   if(status == -1)
   {
      int err = GetLastError();
      // 4014 is common when URL not whitelisted
      PrintFormat("WebRequest failed. Error %d. Did you allow the URL in Options?", err);
      ResetLastError();
      return false;
   }

   if(status < 200 || status >= 300)
   {
      PrintFormat("n8n HTTP status %d. Response headers: %s", status, result_headers);
      return false;
   }

   json_string = CharArrayToString(result);
   // optional: trim whitespace
   StringTrimLeft(json_string);
   StringTrimRight(json_string);

   if(StringLen(json_string) == 0)
      return false;

   return true;
}

//+------------------------------------------------------------------+
//| Parse & execute                                                  |
//+------------------------------------------------------------------+
// --- helper function (put this ABOVE ProcessWebhook)
double NormalizeToDigits(double price, string symbol)
{
   int digits = (int)SymbolInfoInteger(symbol, SYMBOL_DIGITS);
   return NormalizeDouble(price, digits);
}

// --- main webhook processor
void ProcessWebhook(const string json_string)
{
   Print("Received JSON: ", json_string);

   CJAVal json;
   if(!json.Deserialize(json_string))
   {
      Print("JSON Deserialize failed! Raw: ", json_string);
      return;
   }

   string symbol     = json["symbol"].ToStr();
   string action     = json["action"].ToStr();
   double sl_in      = json["stopLoss"].ToDbl();
   double tp_in      = json["takeProfit"].ToDbl();

   if(StringLen(symbol) == 0 || (action != "buy" && action != "sell"))
   {
      Print("Invalid JSON fields: require {symbol, action in ['buy','sell'], stopLoss, takeProfit}");
      return;
   }

   MqlTick tick;
   if(!SymbolInfoTick(symbol, tick))
   {
      Print("Could not get latest price for: ", symbol);
      return;
   }

   // --- symbol params ---
   int    digits     = (int)SymbolInfoInteger(symbol, SYMBOL_DIGITS);
   double point      = SymbolInfoDouble(symbol, SYMBOL_POINT);
   int    stops_lvl  = (int)SymbolInfoInteger(symbol, SYMBOL_TRADE_STOPS_LEVEL);   // in points
   int    freeze_lvl = (int)SymbolInfoInteger(symbol, SYMBOL_TRADE_FREEZE_LEVEL);  // in points
   double minDist    = (stops_lvl + freeze_lvl) * point; // conservative

   double ask = tick.ask;
   double bid = tick.bid;

   // --- validate & adjust SL/TP ---
   double sl = sl_in;
   double tp = tp_in;

   if(action == "buy")
   {
      if(sl <= 0 || sl >= ask - minDist) sl = ask - 2.0 * minDist;
      if(tp <= 0 || tp <= ask + minDist) tp = ask + 2.0 * minDist;
   }
   else // sell
   {
      if(sl <= 0 || sl <= bid + minDist) sl = bid + 2.0 * minDist;
      if(tp <= 0 || tp >= bid - minDist) tp = bid - 2.0 * minDist;
   }

   sl = NormalizeToDigits(sl, symbol);
   tp = NormalizeToDigits(tp, symbol);

   // Log before sending
   PrintFormat("Placing %s %s | ask=%.5f bid=%.5f | SL=%.5f TP=%.5f | minDist=%.5f (pts=%d)",
               action, symbol, ask, bid, sl, tp, minDist, stops_lvl + freeze_lvl);

   bool ok=false;
   if(action == "buy")
      ok = trade.Buy(DefaultLots, symbol, ask, sl, tp, "N8N Signal");
   else
      ok = trade.Sell(DefaultLots, symbol, bid, sl, tp, "N8N Signal");

   if(ok)
      PrintFormat("Trade Executed: %s %s | SL %.5f TP %.5f", action, symbol, sl, tp);
   else
   {
      int err = GetLastError();
      PrintFormat("Trade failed: %s %s | err=%d", action, symbol, err);
      ResetLastError();
   }
}


//+------------------------------------------------------------------+

