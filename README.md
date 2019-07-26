回测模块：
下载数据 按钮   调用self.backtester_engine.start_downloading()方法，参数是 vt_symbol,interval,start,end
Start_downloading方法在cta_backtester的engine中   开启一个线程 跑 run-downloading() 参数一样
run-downloading() 如果history data provided in gateway, then query
else:     data = rqdata_client.query_history(req)
在rqdata.py中 query_history(req)  处理的数据放到data中
 data: List[BarData]  data的数据结构
 存到数据库： database_manager.save_bar_data(data) 
 在database_sql.py中  save_bar_data
  def save_bar_data(self, datas: Sequence[BarData]):
        ds = [self.class_bar.from_bar(i) for i in datas]
        self.class_bar.save_all(ds)    
        
 开始回测  按钮：start_backtesting  调用 self.backtester_engine.start_backtesting
 start_backtesting 方法在cta_backtester的engine中 
 开启一个线程，里面跑run_backtesting方法，参数一样
   engine = self.backtesting_engine
   加载参数 engine.set_parameters，
   engine.add_strategy(strategy_class,setting) 加载策略,和setting
   load_data中 load_bar_data或者load_tick_data
   load_bar_data中database_manager.load_bar_data
   在database_sql.py中load_bar_data
   返回data数据，list[BarData]类型
   给到 self.history_data.extend(data) 里面
   再执行engine.run_backtesting()
   run_backtesting() 在app/strategy/backtesting.py中
   里面 ，func = self.new_bar    函数
   初始化 策略 self.strategy.on_init()
   在AtrRsiStrategy中 on_init（）
   有self.load_bar(10) 加载最近10天数据 在template.py 中
           if not callback:
            callback = self.on_bar
       self.cta_engine.load_bar(self.vt_symbol, days, interval, callback)
      在backtesting中
  /*  在cta_strategy/engine.py中 
    bars=self.query_bar_from_rq(...)或者database_manager.load_bar_data（...)
     for bar in bars:
        callback(bar)
     用on_bar处理每一天bar   //可能看错了   
     on_bar在 atr_rsi_strategy中
     ：  
     self.cancel_all()
     self.am =ArrayManager()  在utility中ArrayManager类 
        am.update_bar(bar)
        把bar的数据 更新到 各种指标array的末端
        再atr_array = am.atr(self.atr_length, array=True)
        计算atr数据，返回result  list或者array类型
      计算指标：  self.atr_value = atr_array[-1]
        self.atr_ma = atr_array[-self.atr_ma_length:].mean()
        self.rsi_value = am.rsi(self.rsi_length)
        下面是这个策略的触发，再报单开仓或者平仓
        发buy(bar.close_price + 5, self.fixed_size) 或者sell(long_stop, abs(self.pos), stop=True)，short,cover,
        buy在template中  buy（..)中send_order  在vt_orderids = self.cta_engine.send_order(...)
         在cta_strategy的engine.py中send_order中有self.send_limit_order(strategy, contract, direction, offset, price, volume, lock)，有发单逻辑    
         在backtesting中 send_limit_order（....)
         self.limit_order_count += 1
         order = OrderData(...)
         self.active_limit_orders[order.vt_orderid] = order
        self.limit_orders[order.vt_orderid] = order
        返回 {'BACKTESTING.1'}
     返回orderids 
    再put_events()  在template.py中
    里面self.cta_engine.put_strategy_event(self)
    在cta_strategy, engine.py中
    put_strategy_event()
    策略先结束
    data 取history_data
    self.callback(data)  用stratery中的on_bar处理data   就是使用策略
    self.strategy.on_start()
   self.output("开始回放历史数据")
        # Use the rest of history data for running backtesting
        for data in self.history_data[ix:]:
            func(data)
  最重要的一步，
  func是 new_bar() 
  里面:
  self.cross_limit_order()
  self.cross_stop_order()
  self.strategy.on_bar(bar)
  self.update_daily_close(bar.close_price)
  cross_limit_order():
  成交限价单 里面有成交逻辑
  cross_stop_order()
  再
  self.strategy.on_bar(bar)  用strategy里面的on_bar处理
  
  update_daily_close(bar.close_price)    //干嘛的
  
  再calculate_result()
        
