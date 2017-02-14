如何与后台对接

1.首先把build varients APP 对应的的mockDebug 改为prodDebug

此时就开始使用后台提供的正式数据了，

但是此时有个问题就是：因为后台提供给我们每个人的API不全，而我们可能同时负责若干个模块，此时从mockDebug 改为prodDebug
会导致我们Mock失效，导致我们某些模块没有了数据，因而发生crash，遇到这种问题只需要将原来mock里面di/module包下面的ServiceModule.Java中提供假数据的方法中copy出mock的假数据到该模块下mvp里对应model中生成数据的方法中即可。
比如

	@Override
    public Observable<List<WareHouse>> getAllWareHouse() {
            List<WareHouse> dataList = new ArrayList<>();
            dataList.add(new WareHouse(1, "仓库A"));
            dataList.add(new WareHouse(2, "仓库B"));
            dataList.add(new WareHouse(3, "仓库C"));
            dataList.add(new WareHouse(4, "仓库D"));
            dataList.add(new WareHouse(5, "仓库E"));
            dataList.add(new WareHouse(6, "仓库F"));
            dataList.add(new WareHouse(7, "仓库G"));
            dataList.add(new WareHouse(8, "仓库H"));
            dataList.add(new WareHouse(9, "仓库I"));
            dataList.add(new WareHouse(10, "尾数仓"));
            dataList.add(new WareHouse(11, "Feeder缓冲区"));

            return Observable.just(dataList);

原来提供真是数据的方法，但是因为对应的API没提供，所以将之前mock数据从module包下面的ServiceModule移动到这里。
	//        return getService().getAllWareHouse().compose(RxsRxSchedulers.<List<WareHouse>>io_main());