### Route Optimizer worker

Pulls from `routres_to_optimize` and does the optimization
sends a update to DB thru the Database API and then deletes the message off the queue
