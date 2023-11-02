```typescript
import {ClerkExpressWithAuth, StrictAuthProp} from '@clerk/clerk-sdk-node';
import {Db, Long, MongoClient, MongoClientOptions} from 'mongodb';
import {Counter, Histogram, register} from 'prom-client';
import express, {NextFunction, Request, Response} from 'express';
import StatusCode from '@instamenta/http-status-codes';
import Vlogger from "@instamenta/vlogger";
import 'dotenv/config';

const http_request_count = new Counter({
name: 'http_requests_total',
help: 'Total number of HTTP requests',
labelNames: ['method', 'route', 'status'],
});
const response_time_histogram = new Histogram({
name: 'http_response_time_seconds',
help: 'Histogram of response times',
labelNames: ['method', 'route', 'status'],
buckets: [0.1, 0.5, 1, 2, 5],
});

function useHttpServer(): express.Express {
const _server = express();

    //* Extensions
    _server.use(require('cors')());
    _server.use(require('helmet')());
    _server.use(require('compression')());
    _server.use(require('cookie-parser')());
    _server.use(require('morgan')('dev'));
    _server.use(express.json());

    //* Prometheus
    _server.use(_metrics_middleware);
    _server.get('/metrics', _metrics_endpoint);

    //* Clerk
    _server.use(ClerkExpressWithAuth({jwtKey: process.env?.CLERK_JWT_PUBLIC_KEY}));

    return _server;
}

function startHttpServer(_server: express.Express) {
const vlogger = Vlogger.getInstance();

    _server.listen(process.env?.PORT, () => {
        console.log('======================================================================');
        vlogger.getVlogger(process.env?.SERVICE_NAME || 'SERVICE').info({
            f: 'startHttpServer',
            m: `[ [ ${process.env?.SERVICE_NAME || 'SERVICE'} ] Running on port: [ ${process.env?.PORT} ]`,
        });
        console.log('======================================================================');
    });
    _server.on('error', (e: Error | unknown) => {
        console.log('======================================================================');
        vlogger.getVlogger(process.env?.SERVICE_NAME || 'SERVICE').error({
            f: 'startHttpServer',
            m: `[ ${process.env?.SERVICE_NAME || 'SERVICE'} ] ran into Error:`,
            e,
        });
        console.log('======================================================================');
    });
}

function useMongoDB(config?: MongoClientOptions | null): { database: Db, db_client: MongoClient } {
Vlogger.getInstance().getVlogger(process.env?.SERVICE_NAME || 'SERVICE')
.info({f: 'initialize_database', m: '[ Connecting to Mongo Client ]'})
const db_client = new MongoClient(process.env?.DB_URI || '', config || {appName: process.env?.SERVICE_NAME,});

    Vlogger.getInstance().getVlogger(process.env?.SERVICE_NAME || 'SERVICE')
        .info({f: 'initialize_database', m: `[ Connecting to Database "${process.env?.DB_NAME || 'main'} ]`})

    return {
        database: db_client.db(process.env?.DB_NAME || 'main'),
        db_client: db_client,
    }

}

class GracefulShutdown {

    public static process_on(_cases_: string[] = ['unhandledRejection', 'uncaughtException']): void {
        _cases_.forEach((_type_: string) => {
            process.on(_type_, (error: Error) => {
                try {
                    Vlogger.getInstance().getVlogger('NODE_PROCESS')
                        .error({
                            e: error,
                            f: 'process_on',
                            m: `[${process.env?.SERVICE_NAME || 'SERVICE'}] ~ process.on: [${_type_}] `
                        })
                } catch {
                    process.exit(1);
                }
            });
        });
    }

    public static process_once(_cases_: string[] = ['SIGTERM', 'SIGINT', 'SIGUSR2']): void {
        _cases_.forEach((_type_: string) => {
            process.once(_type_, (error: Error) => {
                try {
                    Vlogger.getInstance().getVlogger('NODE_PROCESS')
                        .error({
                            e: error,
                            f: 'process_once',
                            m: `[${process.env?.SERVICE_NAME || 'SERVICE'}] - process.on: [${_type_}] `
                        })
                    process.exit(0);
                } finally {
                    process.kill(process.pid, _type_);
                }
            });
        });
    }
}

class ErrorMiddleware {
public static _errorHandler(e: Error, r: Request, w: Response, n: NextFunction) {
Vlogger.getInstance().getVlogger('ErrorMiddleware').error({e, f: '_errorHandler'})
w.status(StatusCode.INTERNAL_SERVER_ERROR)
.json({error: 'Internal Server Error!'}).end();
}

    public static _404Handler(r: Request, w: Response, n: NextFunction) {
        Vlogger.getInstance().getVlogger('ErrorMiddleware').error({e: 'Not Found 404 ', f: '_404Handler', m: r.url})
        w.status(StatusCode.NOT_FOUND)
            .json({error: 'Not Found'}).end();
    }
}

const Exports = {
'GracefulShutdown': GracefulShutdown,
'ErrorMiddleware': ErrorMiddleware,
'useHttpServer': useHttpServer,
'useMongoDB': useMongoDB,
'StatusCode': StatusCode,
'Vlogger': Vlogger,
}

export default Exports;

function _get_elapsed_time(startTime: [number, number]): number {
const elapsedNanoseconds = process.hrtime(startTime);
return elapsedNanoseconds[0] + elapsedNanoseconds[1] * 1e-9;
}

async function _metrics_endpoint(req: Request, res: Response) {
res.set('Content-Type', register.contentType);
res.end(await register.metrics());
}

function _metrics_middleware(req: Request, res: Response, next: NextFunction) {
res.on('finish', () => {
http_request_count.inc({
method: req.method,
route: req.route ? req.route.path : 'unknown',
status: res.statusCode,
});
response_time_histogram.observe({
method: req.method,
route: req.route ? req.route.path : 'unknown',
status: res.statusCode,
}, _get_elapsed_time(process.hrtime()));
});
next();
}

declare global { namespace Express { interface Request extends StrictAuthProp { } } }

// @ts-ignore
BigInt.prototype.toJSON = function () { return this.toString(); };

// @ts-ignore
Long.prototype.toJSON = function () { return this.toString(); };
```