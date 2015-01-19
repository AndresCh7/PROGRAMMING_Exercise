
#include<algorithm>
#include<cstdio>
#include<queue>
#include<vector>
#include<map>
#include<thread>
#include<chrono>
using namespace std;

const int mode_log = 1;
const int mode_stat = 2;
const int mode = mode_stat;

int t;
//Кол-во проезжаемых остановок))
const int deltaN = 4;
//Кол-во мест в автобусе
const int places = 8;
//Интервал прихода пассажиров на остановку:)
const int DeltaT = 7;
//Время ожидания, при котором добавляется новый ав-с
const int maxT = 50;

class Stop;

typedef deque<Stop*>::iterator StopPtr;

class Passenger
{
public:
	Passenger(int time, StopPtr dest) :
		time_come(time), destination(dest){};
	StopPtr destination;
	int time_come;
	int time_load;
};

class Stop
{
public:
	// очередь из пассажиров
	queue<Passenger* > passengers;
	// название остановки
	const string name;
	// расчётное время до следующей
	int timeToNext;
	Stop(const string n, int time) 
		: name(n), timeToNext(time) {};
};




deque <Stop*> stops;//массив из остановок



class Bus
{
public:
	vector <Passenger*> passengers;
};

class Event
{
public:
	Event(int t) : time(t) {};
	int time;
	virtual void process() = 0;
};
// Создаём автобусы
vector<Bus*> buses;

//очередь событий
multimap<int, Event*>events;

class NewPassEvent :public Event
{
public:
	NewPassEvent(int t, StopPtr where_) :Event(t), where(where_){};
	StopPtr where;
	virtual void process()
	{
		auto dest = where;	//придумал куда ему ехать
		for (int i = 0; i < deltaN; i++)
		{
			dest++;
			if (dest == stops.end())
			{
				dest = stops.begin();
			}
		}
		if (mode==mode_log)
			printf("%d A man comes to stop %s\n", t, (*where)->name.c_str());
		//создали нового пассажира:
		Passenger* p = new Passenger (t, dest);
		(*where)->passengers.push(p);//поставили пасс. в очередь на остановке
		
		
		Event* ev = new NewPassEvent(t + DeltaT, where);
		//создали новое событие прихода нового пассажира
		events.insert(pair<int, Event*>(t + DeltaT, ev));
		//добавили это событие в очередь
		
	}
};


//bool arrived(Passenger* p) {
//	return p->destination == p->bus->where;
//}

// событие прихода автобуса
class ArrivalEvent :public Event
{	
public:
	Bus* bus;
	// куда пришёл автобус
	StopPtr where;
	bool operator()(Passenger* p) {
		return p->destination == where;
	}
	ArrivalEvent(int t, StopPtr where_, Bus* b) :
		Event(t), where(where_), bus(b) {};
	virtual void process(){
		if (mode == mode_log)
			printf("%d A bus comes to stop %s\n", t, (*where)->name.c_str());

		remove_if(
			bus->passengers.begin(),
			bus->passengers.end(),
			*this // текущий объект-событие, который может быть вызван как функция
		);
		// пока есть места в автобусе и есть пассажиры на остановке
		while (bus->passengers.size() < places && (!((*where)->passengers.empty()))){
			//выбираем первого пассажира в очереди на остановке:
			auto passenger = (*where)->passengers.front();
			//удаляем выбранного пассажира:
			(*where)->passengers.pop();
			//садим его в автобус:
			bus->passengers.push_back(passenger);
			//отметить время посадки пассажира:
			passenger->time_load = t;
			// вычисляем время ожидания
			if (mode == mode_log)
				printf("%d A passenger waited for %d seconds\n",t,
					passenger->time_load - passenger->time_come
				);
		}
		// вычислить следующую остановку
		auto nextStop = where + 1;
		// если доехал до конца, то начать маршрут с начала
		if (nextStop == stops.end()){
			nextStop = stops.begin();
		}
		//планируем приход автобуса на следующую остановку:
		Event* ev = new ArrivalEvent(
			t + (*where)->timeToNext, nextStop, bus
		);
		// добавить это событие в очередь
		events.insert(pair<int, Event*>(ev->time, ev));
	}
};



class CheckQueueEvent : public Event{
public:
	CheckQueueEvent(int timeNextEvent) : Event(timeNextEvent){}
	virtual void process(){
		for (auto stop : stops){
			if (stop->passengers.size() > 0){
				auto passenger = stop->passengers.front();
				if (t - passenger->time_come > maxT){
					Bus* b = new Bus;
					buses.push_back(b);
					Event* ev = new ArrivalEvent(
						t + 1, stops.begin(), b);
					if (mode == mode_log)
						printf("%d Create new bus\n", t);
					// добавить это событие в очередь
					events.insert(pair<int, Event*>(ev->time, ev));
				}
			}
		}
		Event* ev = new CheckQueueEvent(t + maxT);
		// добавить это событие в очередь
		events.insert(pair<int, Event*>(ev->time, ev));
	}
};

int main()
{
	// создаём остановки
	Stop* mp = new Stop("Mogilevskaya",103);
	stops.push_back(mp);
	stops.push_back(new Stop("Autozavodskaja", 143));
	stops.push_front(new Stop("Autozavodskaja", 163));
	stops.push_back(new Stop("Partyzanskaja", 209));
	stops.push_front(new Stop("Partyzanskaja", 313));
	stops.push_back(new Stop("Traktarny zavod", 233));
	stops.push_front(new Stop("Traktarny zavod", 353));
	stops.push_back(new Stop("Praletarskaja", 128));
	stops.push_front(new Stop("Praletarskaja", 256));
	// планируем события прихода пассажиров
	for (auto it = stops.begin(); it != stops.end(); it++)
	{
		Event* ev = new NewPassEvent(0, it);
		//создали новое событие прихода нового пассажира
		events.insert(pair<int, Event*>(0, ev));
		//добавили это событие в очередь
	}

	//создаем событие проверки очередей
		Event* ev = new CheckQueueEvent(maxT);
	// добавить это событие в очередь
	events.insert(pair<int, Event*>(ev->time, ev));
	
	// номер последнего события, когда была выведена стаитстика
	int lastStat = 0;
	// через сколько событий выводить статистику
	const int statPeriod = 100000;
	// номер текущего события
	int eventNum =0;
	do
	{
		eventNum++;
		auto ev = events.begin();// взять из очереди итератор указывающий на первое событие
		t = ev->first; // установить текущее время
		ev->second->process(); //обработать событие
		delete ev->second; // удалить событие
		events.erase(ev); // удалить ссылку на событие из очереди

		if ((mode == mode_stat) &&
			(eventNum-lastStat > statPeriod) ) {
			printf("\n        time  = %d buses=%d \n\n        Name         queue wait\n",t,buses.size());
			for (auto s : stops) {
				int waiting = (s->passengers.size() > 0) ?
					(t - s->passengers.front()->time_come) :
					0;
				printf("%20s %4d %4d\n",
					s->name.c_str(),
					s->passengers.size(),
					waiting
					);
			}
			std::this_thread::sleep_for(std::chrono::milliseconds(100));

		}
	}
	while (t<10000000);
	return 0;
	


}
