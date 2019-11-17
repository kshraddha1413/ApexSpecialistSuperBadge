# Automate record creation

## Challenge 1
Install the unmanaged package for the schema and stubs for Apex classes and triggers. Rename cases and products to match the
HowWeRoll schema, and assign all profiles to the custom HowWeRoll page layouts for those objects. Use the included package content to automatically create a Routine Maintenance request every time a maintenance request of type Repair or Routine Maintenance is updated to Closed. Follow the specifications and naming 
conventions outlined in the business requirements.

```
trigger MaintenanceRequest on Case (before update, after update) {
    Map<Id,Case> caseMap = new Map<Id,Case>();
    if(Trigger.isUpdate  && Trigger.isAfter){
        for(Case objCase: Trigger.new){
            if (objCase.IsClosed && (objCase.Type.equals('Repair') || objCase.Type.equals('Routine Maintenance'))){
                caseMap.put(objCase.Id, objCase);
            }
        }
        if(!caseMap.values().isEmpty()){
        	MaintenanceRequestHelper.updateWorkOrders(caseMap);    
        }        
    } 
}


```




```
public class MaintenanceRequestHelper {
    
       public static void updateWorkOrders(Map<Id, Case> applicableCases){

	    Map<Id, Integer> prodMap = new Map<Id, Integer>(); 
   		List<Case> uCases = new List<Case>();
        
        List<Product2> listProduct = [select Id, Maintenance_Cycle__c from Product2];       							
		for (Product2 p : listProduct) {
            if (p != null) {
                if(p.Maintenance_Cycle__c != null){
                    prodMap.put(p.Id, Integer.valueOf(p.Maintenance_Cycle__c));
                }               
            }
        }

        for(Case a: applicableCases.values()){
            Case uCase = new Case();
            uCase.Vehicle__c = a.Vehicle__c;
            uCase.Equipment__c = a.Equipment__c;
            uCase.Type = 'Routine Maintenance';
            uCase.Subject = String.isBlank(a.Subject) ? 'Routine Maintenance Request' : a.Subject;
            uCase.Date_Reported__c = Date.today();
            uCase.Status = 'New';
            uCase.Product__c = a.Product__c;
            uCase.AccountId = a.AccountId;
            uCase.ContactId = a.ContactId;
            uCase.AssetId = a.AssetId;
            uCase.Origin = a.Origin;
            uCase.Reason = a.Reason;
            uCase.Date_Due__c =  (prodMap.get(a.Id) != null) ? (Date.today().addDays(Integer.valueOf(prodMap.get(a.Id)))) : (Date.today());
            uCases.add(uCase);
        }
        if(uCases.size() > 0){
            insert uCases;
        }
		        
        
    }    
    
}

```



```
@isTest
public class MaintenanceRequestTest {
	@isTest
    static void testMaintenanceRequest(){
        
        List<Case> cList = new List<Case>();

        Product2 prod = new Product2();
        prod.Cost__c = 5000;
        prod.Name = 'Generator 1000 kW';
        prod.Lifespan_Months__c = 120;
        prod.Maintenance_Cycle__c = 365;
        prod.Current_Inventory__c = 5;
        prod.Replacement_Part__c = false;
        prod.Warehouse_SKU__c = '100003';
        insert prod;
        System.assertEquals(1, [SELECT count() FROM Product2 WHERE Name = 'Generator 1000 kW']);
            
         for(Integer i=1;i<=300;i++) {
            Case caseNew = new Case();
            caseNew.Subject = 'Maintenance';
            caseNew.Type = 'Other';
            caseNew.Status = 'New';
            caseNew.Equipment__c = prod.Id;
            cList.add(caseNew);   
        }
        
        Test.startTest();
        
        insert cList;
        System.assertEquals(300, [SELECT count() FROM Case WHERE Type = 'Other']);
        
        for(Case a : cList){
            a.Type = 'Repair';
            a.Status = 'Closed';
        }
		update cList;
        System.assertEquals(300, [SELECT count() FROM Case WHERE Type = 'Repair']);
        
        Test.stopTest();
        
    }
}
```
#Challenge 2#

```
public class WarehouseCalloutService {
    @future( callout = true )
    public static void runWarehouseEquipmentSync() {
        
        Http http = new Http();
        HttpRequest req = new HttpRequest();
        HTTPResponse res = new HTTPResponse();
        
        req.setEndpoint('https://th-superbadge-apex.herokuapp.com/equipment');
        req.setMethod('GET');
        req.setHeader('Content-Type', 'text-xml');
        res = http.send(req);
        
        List<WarehouseEquipment> eqList = new WarehouseEquipment().parse( res.getBody() );
        List<Product2> prodList = new List<Product2>();
        
       
        for ( WarehouseEquipment w : eqList ) {
            Product2 prod = new Product2( Warehouse_SKU__c  = w.id );
            prod.Replacement_Part__c = true;
            prod.Cost__c = w.cost;
            prod.Current_Inventory__c = w.quantity;
            prod.Lifespan_Months__c = w.lifespan;
            prod.Maintenance_Cycle__c = w.maintenanceperiod;
            prod.Name = w.name; 
            prodList.add(prod);
        }

        upsert prodList;
    }
    
    public class WarehouseEquipment {
        public String name;
        public Boolean replacement;
        public Integer quantity;
        public Integer maintenanceperiod;
        public Integer lifespan;
        public Integer cost;
        public String sku;
        public String id;
        
        public List<WarehouseEquipment> parse( String json ) {
            json.replace( '"id":', '"_id ":' );
            return ( List<WarehouseEquipment> ) System.JSON.deserialize( json, List<WarehouseEquipment>.class );
        }
    }
    
    ```
    #Challenge 3#
    ```
    global class WarehouseSyncSchedule implements Schedulable{
    // implement scheduled code here
    global void execute (SchedulableContext sc){
    WarehouseCalloutService.runWarehouseEquipmentSync();
        String sch = '00 00 01 * * ?';
        String jobId=System.schedule('WarehouseSyncScheduleTest', sch, new WarehouseSyncSchedule());
    }
}
```

#Challenge 4#

```
@isTest
public class WarehouseSyncScheduleTest {
    @isTest static void WarehousescheduleTest(){
        String scheduleTime = '00 00 01 * * ?';
        Test.startTest();
        Test.setMock(HttpCalloutMock.class, new WarehouseCalloutServiceMock());
         String jobID=System.schedule('Warehouse Time To Schedule to Test', scheduleTime, new WarehouseSyncSchedule());
        Test.stopTest();
         CronTrigger a=[SELECT Id FROM CronTrigger where Id =:jobId];
        System.assertEquals(jobID, a.Id,'Schedule');
    }

}
```
```
@isTest
global class WarehouseCalloutServiceMock implements HttpCalloutMock{
    // implement http mock callout
    global HttpResponse respond(HttpRequest request){
        System.assertEquals('https://th-superbadge-apex.herokuapp.com/equipment', request.getEndpoint());
        System.assertEquals('GET', request.getMethod());
        HttpResponse response = new HttpResponse();
        response.setHeader('Content-Type', 'application/json');
        response.setBody('[{"_id":"55d66226726b611100aaf741","replacement":false,"quantity":5,"name":"Generator 1000 kW","maintenanceperiod":365,"lifespan":120,"cost":5000,"sku":"100003"}]');
           response.setStatusCode(200);
       return response;
        
    }
}
```

```

@IsTest
private class WarehouseCalloutServiceTest {
    @isTest
    static void TestWarehouseCalloutService(){
        Test.startTest();
        Test.setMock(HttpCalloutMock.class, new WarehouseCalloutServiceMock());
        WarehouseCalloutService.runWarehouseEquipmentSync();
        Test.stopTest();
    }

}

```
